# Kubernetes Troubleshooting — Production-grade, Interview-ready Guide

This document is a practical, hands-on, senior SRE / Cloud / DevOps level troubleshooting playbook. Each section follows a clear pattern: Symptom → What it means / Root causes → Commands to inspect → Fixes / Mitigations → Real-time example (if applicable). Use this during incident response and interviews.

---

## General Troubleshooting Flow (first 5 minutes)

1. Confirm the symptom and scope
   - Which user, service, or customer is affected? Is it single pod, deployment, node, or cluster-wide?
2. Gather quick facts
   - kubectl get pods -A -o wide --show-labels
   - kubectl get nodes -o wide
   - kubectl get events --sort-by=.lastTimestamp -A | tail -n 100
3. Check the most relevant pod(s)/node(s)
   - kubectl describe pod <pod> -n <ns>
   - kubectl logs <pod> -n <ns> --previous (if CrashLoop)
   - kubectl describe node <node>
4. Isolate the blast radius
   - Which replicas/pods are healthy? Use labels/selectors to list related pods.
5. If production-impacting, create a rollback plan or scale down/up replica counts while debugging.

---

## Useful Commands Cheat Sheet

- Inspect pods: kubectl get pods -n <ns> -o wide
- Describe pod: kubectl describe pod <pod> -n <ns>
- Pod logs (current): kubectl logs <pod> -c <container> -n <ns>
- Pod logs (previous): kubectl logs <pod> -c <container> --previous -n <ns>
- Exec into pod: kubectl exec -it <pod> -c <container> -n <ns> -- /bin/sh
- Events: kubectl get events -n <ns> --sort-by=.lastTimestamp
- Node status: kubectl get nodes -o wide
- Node describe: kubectl describe node <node>
- Resource usage (metrics-server required): kubectl top pod -n <ns> | kubectl top node
- View deployments: kubectl get deploy -n <ns> -o wide
- Check configmaps/secrets: kubectl get cm,secret -n <ns>
- Check CoreDNS: kubectl -n kube-system get pods -l k8s-app=kube-dns
- Check kube-proxy: kubectl -n kube-system get ds kube-proxy -o wide
- Check kubelet journal (node): journalctl -u kubelet -xe

---

## 1) OOMKilled

Symptom:
- Pod status shows `OOMKilled` in `kubectl describe pod` or `kubectl get pod` status.

What it means / Root causes:
- Container exceeded its memory *limit* and the kernel invoked the OOM killer.
- Memory leak in application; wrong memory limits/requests; JVM heap misconfiguration (Xmx > container limit); bursty workloads.

Commands to inspect:
- kubectl describe pod <pod> -n <ns>
- kubectl logs <pod> -n <ns> --previous
- kubectl top pod <pod> -n <ns> (if metrics available)
- On node: dmesg | grep -i oom or journalctl -k | grep -i oom

Fix / Mitigation:
- If requests/limits are too low, increase them. Example: 
  resources:
    requests:
      memory: "512Mi"
    limits:
      memory: "1Gi"
- For JVM apps, set `-Xmx` lower than the container memory limit (leave headroom for non-heap native memory). Example: `-Xmx768m` for a 1Gi limit.
- Identify memory leak via heap dumps / profilers; roll out fix.
- Add vertical scaling (VPA) or horizontal scale-out if stateless.

Real-time example:
- A Java service was being OOMKilled because `-Xmx=2g` while the container limit was `1Gi`. Fix: reduce Xmx to 700m and set limit to 1Gi, or increase limit to 3Gi.

---

## 2) CrashLoopBackOff

Symptom:
- Pod repeatedly fails, restarts and status becomes `CrashLoopBackOff`.

Root causes:
- Application process exits immediately due to startup error
- Missing/invalid configuration, secrets or environment variables
- Init container failure
- Probes (liveness) misconfigured and kill container shortly after start

Commands to inspect:
- kubectl logs <pod> -c <container> --previous -n <ns>
- kubectl describe pod <pod> -n <ns>
- kubectl get events -n <ns>

Fix:
- Read the logs to see the crash trace and fix the root error.
- If due to readiness/liveness probe: increase `initialDelaySeconds` or fix the path/port.
- Validate ConfigMap/Secret mounted correctly; check envs with kubectl exec and env.
- Add `restartPolicy: OnFailure` or appropriate policy if job-like.

Real-time troubleshooting tip:
- Use `kubectl logs --previous` to get the last crash log; `kubectl exec` into a healthy replica to inspect config.

---

## 3) ImagePullBackOff / ErrImagePull

Symptom:
- Pod cannot pull image; events show `Back-off pulling image` or `ErrImagePull`.

Root causes:
- Typo in image name/tag
- Private registry requiring credentials
- Network/DNS issues reaching registry
- Image deleted from registry or manifest not found

Commands to inspect:
- kubectl describe pod <pod> -n <ns> (check events)
- Pull the image manually on a node: docker pull <image> or crictl pull <image>

Fix:
- Correct image name/tag.
- Create image pull secret: 
  kubectl create secret docker-registry regcred --docker-server=<server> --docker-username=<user> --docker-password=<pass> -n <ns>
  and reference via `imagePullSecrets`.
- Ensure nodes can reach the registry (check proxy/firewall).

---

## 4) Pods Evicted / FailedScheduling / Insufficient resources

Symptom:
- Pod phase `Evicted` or scheduler events `FailedScheduling`/`Insufficient memory` or `Insufficient cpu`.

Root causes:
- Node under memory/disk pressure causes eviction
- Requests too high/too low causing bin packing problems
- Taints or lack of tolerations
- No cluster autoscaler to add nodes for pending pods

Commands to inspect:
- kubectl get events --sort-by=.lastTimestamp -A
- kubectl describe pod <pod> -n <ns>
- kubectl describe node <node>
- kubectl top node

Fix:
- Adjust resource requests/limits realistically. Use `requests` to reserve capacity for scheduling.
- Clean node disk (eviction often due to disk pressure). SSH and check `df -h` and kubelet logs.
- Add cluster capacity or enable cluster autoscaler.
- Use PodDisruptionBudget carefully to prevent excessive voluntary disruptions.

Real-time note:
- Rebalancing: drain and cordon nodes then `kubectl drain <node> --ignore-daemonsets --delete-local-data` to migrate workloads during maintenance.

---

## 5) Node NotReady

Symptom:
- Node shows `NotReady` in `kubectl get nodes`

Root causes:
- Kubelet down/crashed or config change
- Network/CNI failure (node cannot reach API or kube-proxy fails)
- DiskPressure or OutOfMemory at node level

Commands to inspect:
- kubectl describe node <node>
- On node: systemctl status kubelet; journalctl -u kubelet -xe
- Check CNI pods: kubectl -n kube-system get pods -o wide

Fix:
- Restart kubelet: systemctl restart kubelet (if control allowed)
- If CNI pods down, inspect CNI pod logs and manifests.
- Remedy disk pressure: free disk space.

---

## 6) Kubelet restarting repeatedly / certificate issues

Symptom:
- Kubelet service crashes or restarts repeatedly.

Root causes:
- Disk full; corrupted kubelet state; expired certs; misconfiguration in kubelet flags.

Commands to inspect:
- journalctl -u kubelet -n 200 --no-pager
- df -h; ls /var/lib/kubelet

Fix:
- Free disk space (journald log rotation, clean container runtime directories)
- Rotate kubelet certificates or rejoin node if certs expired

---

## 7) Networking: Service → Pod path and 502s

Symptom:
- Client gets 502 from ingress/LoadBalancer; service reports endpoints but traffic not reaching pod.

Root causes:
- Pod not listening on expected port or incorrect containerPort
- Service selector mismatch
- NetworkPolicy blocking traffic
- Ingress backend misconfiguration or probe failure

Commands to inspect:
- kubectl get svc -n <ns> -o yaml
- kubectl get endpoints -n <ns>
- kubectl describe ingress <ingress> -n <ns>
- kubectl exec into pod and curl the local port: curl http://127.0.0.1:<port>/health

Fix:
- Fix service selector/port mapping, ensure containerPort matches application.
- Inspect and modify NetworkPolicies to allow traffic from required sources.
- Check ingress controller logs (nginx/contour/traefik).

Real-time example:
- In one incident, ingress returned 502 because the Deployment used `targetPort` 8080 while the container listened on 9090. Updating the Service fixed the issue.

---

## 8) DNS issues inside pods

Symptom:
- Pods cannot resolve service names or external domains.

Root causes:
- CoreDNS pods failing or overloaded
- Wrong /etc/resolv.conf settings (search/path)
- NetworkPolicy or hostNetwork settings interfering

Commands to inspect:
- kubectl -n kube-system get pods -l k8s-app=kube-dns
- kubectl logs -n kube-system <coredns-pod>
- From a pod: nslookup <service> or dig @10.96.0.10 <service> (CoreDNS IP)

Fix:
- Scale CoreDNS (Deployment or ReplicaSet), tune CoreDNS cache/forward settings.
- Ensure cluster DNS IP does not collide with service IPs.

---

## 9) Liveness vs Readiness probes — common mistakes

Symptom:
- App repeatedly restarts or never receives traffic even when healthy.

Root causes:
- Liveness probe too strict or short causing restarts
- Readiness probe failing, removing pod from service discovery

Commands to inspect:
- kubectl describe pod <pod> -n <ns> (inspect probe definitions and recent probe failures)

Fix:
- Increase `initialDelaySeconds`, `periodSeconds`, or `timeoutSeconds` based on app startup time.
- Prefer readiness probe that checks application readiness (DB connections, caches warmed) not just process alive.

Example probe config:
- livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10

---

## 10) Scheduling, Taints & Tolerations, Affinity

- Use `kubectl describe node` and `kubectl get pod -o yaml` to see which taints block scheduling.
- Add tolerations to pods or remove taints from nodes. Example: `kubectl taint node node1 key=value:NoSchedule-` to remove.
- Use nodeAffinity to pin workloads to nodes and podAntiAffinity to spread replicas.

---

## 11) StatefulSet vs Deployment (short interview-ready notes)

- Deployment: best for stateless workloads. Pods are fungible, scale quickly, no stable network identity.
- StatefulSet: stable pod identity (sticky names), stable storage (Per-Pod PVCs), ordered rolling updates. Use for databases, Kafka, Zookeeper.

---

## 12) Upgrade & Maintenance Procedures (safe steps)

1. Drain node: kubectl drain <node> --ignore-daemonsets --delete-local-data\2. Upgrade control plane via managed provider console (AKS/GKE/EKS) or kubeadm for self-managed clusters.
3. Upgrade node pool: rotate node pool with new version, cordon/drain and uncordon.
4. Run smoke tests and verify telemetry/alerts.

Always have rollback steps and backups for persistent workloads.

---

## 13) Observability & Prevention

- Metrics: Prometheus + kube-state-metrics + node-exporter. Set alerts for OOM, nodeNotReady, high crashloop rate.
- Logs: Centralize with EFK/PLG stacks. Enable structured logs and correlate via trace IDs.
- Tracing: OpenTelemetry, Jaeger.
- Automated tests: readiness/liveness integration tests in CI; canary deployments for risky changes.
- Resource policies: use LimitRanges, PodSecurityPolicies (or Pod Security Admission) and network policies.

---

## 14) Incident Playbook Snippets (fast actions)

- If production pod OOMKilled: scale up replicas or deploy a temporary higher-limit version, then revert after fix.
- If Service 502: curl the pod, check service endpoints, correct port mapping, and redeploy.
- If many nodes NotReady: check control plane endpoints, etcd, and kubelet logs; consider draining cluster for maintenance.

---

## 15) Helpful Debugging Patterns and Tips

- Reproduce locally: run the container image locally and exercise with same envs.
- Use ephemeral debug containers: kubectl debug -it pod/<pod> --image=busybox --target=<container>
- When pods are pending, look at `kubectl get events` first — it usually tells why.
- Prefer small, incremental changes when debugging production.
- Capture full context: pod YAML, events, node state, logs, and monitoring graphs before applying fixes — useful for post-incident analysis.

---

## Appendix: Example Resource YAMLs

Example resources for common fixes:

1) Increase memory limits - Deployment snippet
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: example:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

2) Liveness/Readiness example
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  timeoutSeconds: 5
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 3
```

---

End of document. Keep this file as the canonical, interview-ready troubleshooting reference and expand it with organization-specific operational runbooks, alerts thresholds, and contact procedures.
