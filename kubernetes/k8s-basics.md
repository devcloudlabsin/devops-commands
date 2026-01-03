# Kubernetes Admin & Developer Commands — Architect Level
This README is a comprehensive, practical reference of Kubernetes commands and patterns for admins and developers — from basic to advanced/pro. Each section contains commands, detailed explanations of flags and syntax, examples, and verification steps. Use this as an operational playbook and learning guide.

Table of contents
- Prerequisites & conventions
- kubectl fundamentals (config, contexts)
- Namespaces & resource scoping
- Pods: create, inspect, exec, logs, copy
- Controllers: Deployments, ReplicaSets, DaemonSets, StatefulSets
- Jobs & CronJobs
- Services & networking (ClusterIP, NodePort, LoadBalancer)
- Ingress and Ingress Controllers
- ConfigMaps & Secrets
- Volumes, PersistentVolumes, PVCs, StorageClass
- Nodes, kubelet, kubeadm, cluster lifecycle
- Scheduling: nodeSelector, affinities, taints/tolerations
- RBAC: Roles, ClusterRoles, bindings, auth troubleshooting
- Autoscaling: HPA, VPA, Cluster Autoscaler
- Rolling updates, rollbacks, strategies
- CRDs & Operators
- Security: Pod Security, NetworkPolicy, image scanning, runtime
- Debugging & Troubleshooting: logs, describe, events, exec, debug pods
- Observability: metrics, kubectl top, port-forwarding, debug ports
- Helm & package management
- Advanced admin operations & API server interactions
- Useful kubectl flags, output formatting, and tips
- Appendix: YAML snippets & cheat sheet

Prerequisites & conventions
- kubectl >= 1.22 recommended (commands vary by version; confirm compatibility).
- You must have kubeconfig access to clusters. Default file: $KUBECONFIG or ~/.kube/config.
- Commands are shown with explanations. Use `--namespace` (or `-n`) to scope to a namespace. Replace placeholders like <pod>, <ns>, <file.yaml>.
- Many commands show both short and long flag forms.
- Verification commands are included under each section.

kubectl fundamentals (config, contexts)
- List clusters/contexts in kubeconfig:
  ```
  kubectl config view --minify      # shows current context
  kubectl config get-contexts       # list contexts
  kubectl config current-context    # show active context
  ```
  Explanation:
  - `kubectl config view` prints the kubeconfig. `--minify` shows only info for current context.
  - `get-contexts` lists name, cluster, auth info.
- Switch context:
  ```
  kubectl config use-context <context-name>
  ```
  Verification:
  ```
  kubectl config current-context
  kubectl get nodes
  ```
- Add a cluster/user/context (example):
  ```
  kubectl config set-cluster my-cluster --server=https://1.2.3.4:6443 --certificate-authority=/path/ca.crt
  kubectl config set-credentials my-user --client-certificate=/path/client.crt --client-key=/path/client.key
  kubectl config set-context my-context --cluster=my-cluster --user=my-user --namespace=my-namespace
  kubectl config use-context my-context
  ```

Namespaces & resource scoping
- Create a namespace:
  ```
  kubectl create namespace my-ns
  # or from YAML
  kubectl apply -f ns.yaml
  ```
  ns.yaml:
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: my-ns
  ```
- List resources in a namespace:
  ```
  kubectl get all -n my-ns        # all common resources (pods,svc,deploy,rs)
  kubectl get pods -n my-ns -o wide
  ```
- Delete namespace and wait:
  ```
  kubectl delete namespace my-ns
  kubectl get namespace my-ns -o yaml   # check deletionTimestamp
  ```
Verification:
```
kubectl get namespaces
kubectl get pods -A      # shows pods across all namespaces
```

Pods
- Create a pod (imperative):
  ```
  kubectl run nginx --image=nginx:1.25 --restart=Never -n my-ns
  ```
  Explanation:
  - `--restart=Never` creates a Pod (not a Deployment).
- Create from YAML:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox-sleep
    namespace: my-ns
  spec:
    containers:
    - name: busybox
      image: busybox:1.36
      command: ["sleep", "3600"]
  ```
  Apply:
  ```
  kubectl apply -f busybox-pod.yaml
  ```
- Inspect pod:
  ```
  kubectl get pods -n my-ns
  kubectl describe pod busybox-sleep -n my-ns
  ```
  - `describe` includes events, conditions, container statuses, node assignment, mounts.
- Exec into a running container:
  ```
  kubectl exec -it busybox-sleep -n my-ns -- sh
  ```
  - `-c <container-name>` if multiple containers.
- Stream logs:
  ```
  kubectl logs pod-name -n my-ns            # logs of main container
  kubectl logs pod-name -c container -n ns  # container-specific
  kubectl logs -f pod-name -n ns            # follow
  kubectl logs --since=1h pod-name -n ns    # since 1 hour
  ```
- Copy files:
  ```
  kubectl cp localfile my-ns/pod-name:/tmp/remote -c container
  ```
Verification:
```
kubectl get pod busybox-sleep -n my-ns -o wide
kubectl logs busybox-sleep -n my-ns
kubectl exec -it busybox-sleep -n my-ns -- ls /
```

Deployments, ReplicaSets, DaemonSets, StatefulSets
- Create Deployment (declarative):
  deployment.yaml:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deploy
    namespace: my-ns
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: nginx
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.25
          ports:
          - containerPort: 80
  ```
  ```
  kubectl apply -f deployment.yaml
  kubectl get deployments -n my-ns
  kubectl get rs -n my-ns
  kubectl get pods -l app=nginx -n my-ns
  ```
- Update image (imperative):
  ```
  kubectl set image deployment/nginx-deploy nginx=nginx:1.26 -n my-ns
  ```
  Verification:
  ```
  kubectl rollout status deployment/nginx-deploy -n my-ns
  kubectl rollout history deployment/nginx-deploy -n my-ns
  kubectl describe deployment nginx-deploy -n my-ns
  kubectl get pods -l app=nginx -o wide -n my-ns
  ```
- Rollback:
  ```
  kubectl rollout undo deployment/nginx-deploy -n my-ns
  # or to a specific revision:
  kubectl rollout undo deployment/nginx-deploy --to-revision=2 -n my-ns
  ```
- DaemonSet (one pod per node matching selectors):
  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: node-exporter
    namespace: kube-system
  spec:
    selector:
      matchLabels:
        app: node-exporter
    template:
      metadata:
        labels:
          app: node-exporter
      spec:
        containers:
        - name: exporter
          image: prom/node-exporter:1.5.0
  ```
  ```
  kubectl apply -f daemonset.yaml
  kubectl get daemonset -n kube-system
  ```
- StatefulSet (stable network identity & storage):
  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra
    namespace: my-ns
  spec:
    serviceName: "cassandra"
    replicas: 3
    selector:
      matchLabels:
        app: cassandra
    template:
      metadata:
        labels:
          app: cassandra
      spec:
        containers:
        - name: cassandra
          image: cassandra:4.0
  ```
  Verification:
  ```
  kubectl get statefulset -n my-ns
  kubectl get pods -l app=cassandra -n my-ns -o wide
  kubectl describe pod cassandra-0 -n my-ns
  ```
Jobs & CronJobs
- Job (one-off):
  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: pi
  spec:
    template:
      spec:
        containers:
        - name: pi
          image: perl
          command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(20)"]
        restartPolicy: Never
    backoffLimit: 4
  ```
  ```
  kubectl apply -f job.yaml
  kubectl get jobs
  kubectl logs job/pi
  kubectl describe job/pi
  ```
- CronJob:
  ```yaml
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: hello
  spec:
    schedule: "*/5 * * * *"
    jobTemplate:
      spec:
        template:
          spec:
            containers:
            - name: hello
              image: busybox
              args: ["echo", "Hello from CronJob"]
            restartPolicy: OnFailure
  ```
  ```
  kubectl get cronjob
  kubectl get jobs --watch
  ```

Services & networking
- Service types:
  - ClusterIP (internal)
  - NodePort (exposes port on nodes)
  - LoadBalancer (cloud LB)
- Create ClusterIP service:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-svc
    namespace: my-ns
  spec:
    selector:
      app: nginx
    ports:
    - protocol: TCP
      port: 80
      targetPort: 80
    type: ClusterIP
  ```
  ```
  kubectl apply -f svc.yaml
  kubectl get svc -n my-ns
  kubectl describe svc nginx-svc -n my-ns
  ```
- Create NodePort (imperative):
  ```
  kubectl expose deployment/nginx-deploy --type=NodePort --name=nginx-nodeport -n my-ns --port=80 --target-port=80
  kubectl get svc nginx-nodeport -n my-ns -o yaml
  ```
  - NodePort exposes a port in range 30000–32767 by default. Use `--node-port=31000` to pick one.
- LoadBalancer:
  ```
  kubectl expose deployment/nginx-deploy --type=LoadBalancer --name=nginx-lb -n my-ns
  kubectl get svc nginx-lb -n my-ns --watch
  ```
Verification:
```
kubectl get svc -n my-ns
kubectl describe svc nginx-lb -n my-ns
# Curl service from inside cluster:
kubectl run curlpod --image=radial/busyboxplus:curl -i --tty --rm --restart=Never -- /bin/sh -c "curl -s http://nginx-svc.my-ns.svc.cluster.local"
```

Ingress & Ingress Controllers
- Ingress resource (example for nginx-ingress controller):
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
    namespace: my-ns
    annotations:
      kubernetes.io/ingress.class: "nginx"
  spec:
    rules:
    - host: example.company.internal
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nginx-svc
              port:
                number: 80
  ```
  Apply and verify:
  ```
  kubectl apply -f ingress.yaml
  kubectl get ingress -n my-ns
  kubectl describe ingress my-ingress -n my-ns
  ```
  Notes:
  - Ingress requires a controller (NGINX, contour, Istio Gateway, AWS ALB, etc).
  - Annotations control controller-specific features (tls, rewrites, rate limits).

ConfigMaps & Secrets
- ConfigMap (from literal / file):
  ```
  kubectl create configmap my-config --from-literal=LOG_LEVEL=debug -n my-ns
  kubectl create configmap my-config --from-file=app.properties -n my-ns
  ```
  YAML:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: my-config
  data:
    LOG_LEVEL: debug
  ```
  Mount into pod or use as env:
  ```yaml
  envFrom:
  - configMapRef:
      name: my-config
  ```
- Secrets:
  - Create from literal:
    ```
    kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password='S3cr3t' -n my-ns
    ```
  - From file (binary-safe):
    ```
    kubectl create secret generic tls-secret --from-file=tls.crt=./tls.crt --from-file=tls.key=./tls.key -n my-ns
    ```
  - Use as env or volume; example env:
    ```yaml
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    ```
  Security note:
  - Secrets are base64-encoded in etcd by default; enable encryption at rest for etcd! (see Security section)
Verification:
```
kubectl get configmap my-config -o yaml -n my-ns
kubectl describe secret db-secret -n my-ns
kubectl get secret db-secret -o jsonpath="{.data.username}" | base64 --decode
```

PersistentVolumes, PVCs, StorageClass
- StorageClass (dynamic provisioner):
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast-ssd
  provisioner: kubernetes.io/aws-ebs   # example for AWS
  volumeBindingMode: WaitForFirstConsumer
  parameters:
    type: gp3
  ```
- PersistentVolumeClaim:
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: data-pvc
    namespace: my-ns
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 20Gi
  ```
- Use PVC in pod:
  ```yaml
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: data
      mountPath: /var/lib/data
  ```
Verification:
```
kubectl get pvc -n my-ns
kubectl describe pvc data-pvc -n my-ns
kubectl get pv
```

Nodes, kubelet, kubeadm, cluster lifecycle
- List nodes and status:
  ```
  kubectl get nodes -o wide
  kubectl describe node <node-name>
  kubectl get cs         # componentstatus (deprecated)
  ```
- Drain a node for maintenance:
  ```
  kubectl drain <node> --ignore-daemonsets --delete-local-data --force
  # After maintenance:
  kubectl uncordon <node>
  ```
  Explanation:
  - `--ignore-daemonsets` keeps DaemonSet pods (node-exporter, kube-proxy).
  - `--delete-local-data` drains pods with local storage (use with care).
- kubeadm init (control plane bootstrap, example):
  ```
  kubeadm init --control-plane-endpoint "lb.example:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
  ```
  - This prints join commands for control-plane and worker nodes.
- Join a worker:
  ```
  kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
  ```
- Upgrade control plane:
  ```
  kubeadm upgrade plan
  kubeadm upgrade apply v1.26.2
  ```
Verification:
```
kubectl get nodes
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-<node>         # via node or journalctl on host
```

Scheduling: nodeSelector, affinities, taints/tolerations
- nodeSelector example (simple):
  ```yaml
  spec:
    nodeSelector:
      node-role.kubernetes.io/worker: ""
  ```
- node affinity (preferred or required):
  ```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  ```
- Pod affinity/anti-affinity (spread workloads):
  ```yaml
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: "kubernetes.io/hostname"
  ```
- Taints & tolerations:
  - Taint a node:
    ```
    kubectl taint nodes node1 key=value:NoSchedule
    ```
  - Tolerate in pod:
    ```yaml
    tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
    ```
Verification:
```
kubectl describe node node1 | grep -i taint -A 3
kubectl get pods -o wide -n my-ns    # check node placement
```

RBAC: Roles, ClusterRoles, Bindings
- Create Role (namespace-scoped):
  ```yaml
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    namespace: my-ns
    name: pod-reader
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  ```
- Bind Role to a user/serviceaccount:
  ```yaml
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: read-pods
    namespace: my-ns
  subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: my-ns
  roleRef:
    kind: Role
    name: pod-reader
    apiGroup: rbac.authorization.k8s.io
  ```
- Test access:
  ```
  kubectl auth can-i get pods --as=system:serviceaccount:my-ns:my-sa -n my-ns
  kubectl auth can-i create deployments --namespace=my-ns
  ```
Verification:
```
kubectl get rolebinding -n my-ns
kubectl auth can-i --list --namespace=my-ns
```

Autoscaling
- Horizontal Pod Autoscaler (HPA) (v2 uses metrics):
  ```
  kubectl autoscale deployment nginx-deploy --cpu-percent=50 --min=2 --max=10 -n my-ns
  kubectl get hpa -n my-ns
  ```
  YAML (v2):
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: hpa-nginx
    namespace: my-ns
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-deploy
    minReplicas: 2
    maxReplicas: 10
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  ```
- Vertical Pod Autoscaler (VPA) — adjust pod resource requests (controller must be installed).
- Cluster Autoscaler (cloud-specific) — scale node groups when pods unschedulable.
Verification:
```
kubectl get hpa -n my-ns
kubectl describe hpa hpa-nginx -n my-ns
kubectl top pods -n my-ns    # metrics-server must be installed
```

Rolling updates, rollbacks, strategies
- Rolling update via Deployment (see Deployment strategy fields earlier).
- Check rollout status:
  ```
  kubectl rollout status deployment/nginx-deploy -n my-ns
  kubectl rollout history deployment/nginx-deploy -n my-ns
  kubectl rollout undo deployment/nginx-deploy -n my-ns
  ```
- Canary deployments (manual approach):
  - Create new deployment with small replica count, route subset of traffic via service/ingress to canary.
- Blue/Green:
  - Maintain two deployments/services; switch service selector to new set after testing.

CRDs & Operators
- CRD create:
  ```yaml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    name: widgets.example.com
  spec:
    group: example.com
    names:
      kind: Widget
      plural: widgets
      singular: widget
    scope: Namespaced
    versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
  ```
  ```
  kubectl apply -f crd.yaml
  kubectl get crd
  ```
- Operators:
  - Use Operator SDK or Helm-based Operators to encapsulate app lifecycle logic. Interact with CRs via kubectl like any resource:
    ```
    kubectl get widgets -n my-ns
    kubectl describe widget my-widget -n my-ns
    ```
Verification:
```
kubectl get crds
kubectl get <plural> -A
```

Security: Pod Security, NetworkPolicy, image scanning, runtime
- Pod Security Admission (PSA) — built-in since k8s 1.22:
  - Enforce modes: `enforce`, `audit`, `warn`.
  - Use PodSecurityLabels on namespaces:
    ```yaml
    apiVersion: policy/v1
    kind: PodSecurityPolicy    # deprecated in recent k8s, prefer PSA or Gatekeeper
    ```
  - Example PSA label (namespace):
    ```
    kubectl label namespace my-ns pod-security.kubernetes.io/enforce=restricted
    ```
- NetworkPolicy (deny-by-default within namespace if implemented by CNI):
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: deny-all
    namespace: my-ns
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
    - Egress
  ```
  - Allow only specific traffic:
    ```yaml
    spec:
      podSelector:
        matchLabels:
          role: db
      ingress:
      - from:
        - podSelector:
            matchLabels:
              role: backend
  ```
- Image scanning and admission:
  - Use tools: Trivy, Clair, Aqua, Twistlock + admission controllers (OPA Gatekeeper, Kyverno).
- Secrets encryption in etcd:
  - Configure Kubernetes API server `--encryption-provider-config` and enable encryption for secrets.
Verification:
```
kubectl get networkpolicies -n my-ns
kubectl get pods -n my-ns -o=jsonpath='{.items[*].spec.containers[*].securityContext}'
kubectl auth can-i create podsecuritypolicies   # if PSP is enabled
```

Debugging & Troubleshooting
- Events:
  ```
  kubectl get events -n my-ns --sort-by='.metadata.creationTimestamp'
  ```
- Describe shows events & status:
  ```
  kubectl describe pod <pod> -n my-ns
  ```
- Logs:
  ```
  kubectl logs pod -c container -n my-ns
  kubectl logs -l app=nginx -n my-ns --tail=100   # aggregate logs from pods with label
  ```
- Exec into container and run diagnostic commands:
  ```
  kubectl exec -it pod -- sh
  ```
- Debug a failing pod with ephemeral debug container (kubectl debug, requires server support):
  ```
  kubectl debug -it pod/<pod-name> -n my-ns --image=busybox --target=<pod-name>
  ```
- Port forward to access service locally:
  ```
  kubectl port-forward svc/nginx-svc 8080:80 -n my-ns
  curl http://localhost:8080
  ```
- Check scheduling issues:
  ```
  kubectl get events --field-selector involvedObject.kind=Pod -n my-ns
  kubectl describe pod <pod> -n my-ns | grep -i unschedul
  kubectl top nodes
  ```
- Check node logs on host:
  - Use SSH and journalctl:
    ```
    sudo journalctl -u kubelet -f
    sudo journalctl -u docker -f   # or containerd
    ```
- Debugging crash loops:
  ```
  kubectl describe pod <crashpod> -n my-ns   # Look for CrashLoopBackoff reasons
  kubectl logs <crashpod> -c <container> --previous -n my-ns   # logs from previous instance
  ```
- Get core dumps & exec into paused containers using `kubectl attach` / `kubectl debug`.

Observability: metrics, kubectl top, port-forwarding
- Install metrics-server for `kubectl top`.
- Use:
  ```
  kubectl top nodes
  kubectl top pods -n my-ns
  ```
- Port-forward Prometheus/Grafana:
  ```
  kubectl port-forward svc/prometheus 9090:9090 -n monitoring
  kubectl port-forward svc/grafana 3000:3000 -n monitoring
  ```
- Debugging services with temporary pods (curl/test):
  ```
  kubectl run -it --rm --image=radial/busyboxplus:curl debug -- /bin/sh
  # then curl svc.namespace.svc.cluster.local
  ```

Helm & package management
- Install Helm chart:
  ```
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo update
  helm install my-postgres bitnami/postgresql --namespace my-ns --create-namespace --set global.postgresql.postgresqlPassword=$PGPASS
  ```
- Upgrade & rollback:
  ```
  helm upgrade my-postgres bitnami/postgresql -n my-ns --set replicaCount=2
  helm history my-postgres -n my-ns
  helm rollback my-postgres 1 -n my-ns
  ```
Verification:
```
helm ls -n my-ns
kubectl get pods -n my-ns -l app.kubernetes.io/instance=my-postgres
```

Advanced admin operations & API server interactions
- Directly query Kubernetes API via kubectl:
  ```
  kubectl get --raw "/api/v1/namespaces" | jq
  kubectl proxy &   # start local proxy at http://127.0.0.1:8001
  curl localhost:8001/api/v1/namespaces
  ```
- Edit resources (live edit):
  ```
  kubectl edit deployment nginx-deploy -n my-ns
  ```
- Apply YAML with server-side apply (SSA):
  ```
  kubectl apply --server-side -f resource.yaml
  kubectl apply -f resource.yaml --record
  ```
  Notes:
  - SSA uses managedFields; better for controllers & GitOps.
- Garbage collection & ownerReferences:
  - When creating resources that should be deleted when parent deleted, set ownerReferences.
- API discovery:
  ```
  kubectl api-resources
  kubectl api-versions
  ```

Useful kubectl flags, output formatting, and tips
- Output formats:
  ```
  -o wide       # extra columns
  -o yaml
  -o json
  -o jsonpath='{.items[*].metadata.name}'
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
  ```
- Watch resources:
  ```
  kubectl get pods -n my-ns --watch
  kubectl get events -w
  ```
- Label & annotate:
  ```
  kubectl label pod mypod env=prod -n my-ns --overwrite
  kubectl annotate deployment nginx owner=team-a -n my-ns --overwrite
  ```
- Patching resources (strategic merge, JSON patch):
  ```
  kubectl patch deployment nginx-deploy -p '{"spec": {"replicas": 5}}' -n my-ns
  kubectl patch svc my-svc --type='json' -p='[{"op":"replace","path":"/spec/type","value":"NodePort"}]' -n my-ns
  ```
- Delete with grace period and cascading:
  ```
  kubectl delete pod mypod -n my-ns --grace-period=0 --force
  kubectl delete deployment nginx-deploy -n my-ns --cascade=foreground
  ```
- Bulk operations by label:
  ```
  kubectl delete pods -l app=nginx -n my-ns
  kubectl scale deployment -l app=nginx --replicas=0 -n my-ns
  ```
- Dry run:
  ```
  kubectl apply -f resource.yaml --dry-run=client
  kubectl create -f resource.yaml --dry-run=server -o yaml
  ```

Appendix: YAML snippets & cheat sheet
- Minimal deployment:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata: { name: app-deploy }
  spec:
    replicas: 2
    selector: { matchLabels: { app: app } }
    template:
      metadata: { labels: { app: app } }
      spec:
        containers:
        - name: app
          image: my-app:latest
          ports: [{ containerPort: 80 }]
  ```
- Minimal service:
  ```yaml
  apiVersion: v1
  kind: Service
  metadata: { name: app-svc }
  spec:
    selector: { app: app }
    ports: [{ protocol: TCP, port: 80, targetPort: 80 }]
    type: ClusterIP
  ```

Quick verification checklist (pre-flight)
- kubectl version --client && kubectl version
- kubectl cluster-info
- kubectl get nodes
- kubectl get pods -A
- kubectl get events -A --sort-by=.metadata.creationTimestamp
- Ensure metrics-server / ingress controller / CNI are installed for relevant features.

Best practices & tips
- Use declarative YAMLs stored in Git (GitOps).
- Use namespaces for separation and RBAC scope.
- Prefer Deployments & StatefulSets over bare Pods for production.
- Use readiness & liveness probes carefully.
- Use resource requests & limits for stable scheduling.
- Encrypt secrets in etcd and use external secret stores for production (e.g., Vault).
- Automate backups of etcd and test restore procedures regularly.
- Monitor control plane components and etcd.
- Use network policies to limit lateral movement.
- Regularly run `kubectl auth can-i` checks for least privilege audits.

Further learning and tools
- kubectl plugins: krew plugin manager (krew), e.g., `krew install ctx` for context switching.
- Debugging aids: kubectl-debug, stern (logs), k9s (terminal UI).
- Observability: Prometheus + Grafana, Loki for logs, Jaeger for tracing.
- Policy: OPA Gatekeeper, Kyverno.
