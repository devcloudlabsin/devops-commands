# Helm — From Basics to Pro (End-to-End Commands, Explanations & Troubleshooting)

This README is an operational and learning playbook for Helm (v3+), covering beginner → advanced → pro topics with examples, verification steps, and troubleshooting. It is intended for platform engineers, cluster admins and developers who manage applications using Helm charts.

Table of contents
- Prerequisites & Conventions
- Quick Helm Concepts
- Installation & Setup
- Repositories & Charts (search, add, update, index)
- Installing, Upgrading & Uninstalling Releases
- Values, Overrides & Precedence
- Template Rendering & Debugging
- Chart Authoring: structure, Chart.yaml, values, templates
- Chart Dependencies & Umbrella Charts
- Hooks, Tests & CRDs
- OCI registries & chart publishing
- Signing, provenance, and chart security
- Plugins & recommended tooling
- CI/CD patterns (GitHub Actions example, GitOps)
- Helmfile / Chartfile workflows
- Best practices & production checklist
- Troubleshooting common issues & commands
- Appendix: quick command cheat sheet

Prerequisites & conventions
- Helm v3.x+ (Helm 3 removed Tiller; commands assume Helm 3 or newer).
- kubectl configured and working against the target cluster.
- Replace placeholders like <chart>, <release>, <namespace>, <values.yaml>, <repo-url>, <registry>, <key> with your values.
- Commands include long/short flags where applicable. Use `--namespace` or `-n` to scope operations.

Quick Helm Concepts
- Chart: packaged Kubernetes resources (templates + defaults) representing an application.
- Release: an installed instance of a chart in a cluster (name + namespace).
- Values: user-supplied configuration to customize installed templates.
- Template rendering: Helm uses Go templates + Sprig functions to render chart templates into Kubernetes manifests.
- Repositories: HTTP/OCI registries where charts are stored.

Installation & Setup
- Install Helm (Linux/macOS with script or package manager):
  ```bash
  # Homebrew (macOS/Linux)
  brew install helm

  # Script (Linux)
  curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  ```
- Verify:
  ```bash
  helm version       # client version (and server for v2)
  kubectl version
  ```

Repositories & Charts
- Add a chart repository:
  ```bash
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo update
  helm repo list
  ```
  Explanation:
  - `helm repo add NAME URL` registers the repo locally.
  - `helm repo update` refreshes chart index caches.
- Search charts:
  ```bash
  helm search repo nginx        # search local cache of repos
  helm search hub mysql         # search Artifact Hub
  ```
- Inspect chart info:
  ```bash
  helm show chart bitnami/nginx           # Chart.yaml content
  helm show values bitnami/nginx          # default values
  helm show readme bitnami/nginx          # README.md
  ```
- Download / fetch a chart:
  ```bash
  helm pull bitnami/nginx --untar --version 9.3.0
  ```
  Verification:
  ```bash
  ls nginx/            # chart directory if --untar
  helm show chart ./nginx
  ```

Installing, Upgrading & Uninstalling Releases
- Install a chart:
  ```bash
  helm install my-nginx bitnami/nginx -n web --create-namespace
  ```
  Explanation:
  - `install <release-name> <chart>`.
  - If `<release-name>` omitted, Helm will generate one (not recommended for production).
- Upgrade or install if missing:
  ```bash
  helm upgrade --install my-nginx bitnami/nginx -n web --create-namespace
  ```
  Useful flags explained:
  - `--values, -f`: supply values file (can pass multiple).
  - `--set`: set/override specific values (see Values precedence below).
  - `--atomic`: roll back on failure automatically.
  - `--wait`: wait until all resources are ready (pods, PVCs, services).
  - `--timeout`: set wait timeout (e.g., `--timeout 10m`).
  - `--reuse-values`: keep previous release values when upgrading (useful for patch upgrades).
  - `--reset-values`: ignore previous release values.
- Check release status:
  ```bash
  helm list -n web
  helm status my-nginx -n web
  helm history my-nginx -n web
  helm get all my-nginx -n web        # manifests, hooks, notes, values
  helm get values my-nginx -n web
  ```
- Uninstall a release:
  ```bash
  helm uninstall my-nginx -n web
  ```
  - Keep history (default behavior keeps release history in cluster until purged).
  Verification:
  ```bash
  kubectl get pods -n web
  helm list -n web
  ```

Values, Overrides & Precedence
- Values precedence (highest → lowest):
  1. `--set` / `--set-string` / `--set-file`
  2. Values files passed with `-f` (later files override earlier files)
  3. Chart's `values.yaml`
  4. Chart `defaults` baked in templates (rare)
- Set examples:
  ```bash
  helm install app ./chart -n prod \
    --set image.tag=1.2.3 \
    --set service.type=NodePort \
    -f prod-values.yaml \
    -f prod-secrets.yaml
  ```
  - `--set-string` forces string type (useful when numeric/string ambiguity exists).
  - `--set-file` allows injecting file content (e.g., TLS cert).
  - `--set-json` / `--set-literal` (depends on helm version/plugins) — be careful with arrays and complex structures; prefer values files for complex nested configs.
- Inspect final merged values for a release:
  ```bash
  helm get values my-nginx -n web --all    # includes computed/defaults if --all supported
  ```
  To render templates locally with values:
  ```bash
  helm template my-nginx ./chart -f my-values.yaml --namespace web
  ```

Template Rendering & Debugging
- Render templates locally (no cluster):
  ```bash
  helm template my-release ./my-chart -f values.yaml
  ```
- Lint chart:
  ```bash
  helm lint ./my-chart
  ```
  Explanation:
  - `helm lint` checks chart structure and templates (basic validation).
- Use `--debug` for more output:
  ```bash
  helm install my-nginx bitnami/nginx -n web --debug --dry-run
  ```
  - `--dry-run` + `--debug` will show rendered manifests and template errors without installing.
- Useful template functions & patterns:
  - `{{ .Values }}`, `{{ .Chart }}`, `{{ .Release }}`, `{{ .Capabilities }}`, `{{ include "chart.helpers" . }}`, `{{ tpl }}` to evaluate templated values.
  - Helpers file: `_helpers.tpl` for reusable template functions and label generation.
- Common rendering gotchas:
  - Indentation and `toYaml` / `nindent` usage when embedding subobjects into YAML blocks.
  - Missing required value — use `required` function:
    ```yaml
    image:
      repository: {{ required "image.repository is required" .Values.image.repository }}
    ```
  Verification:
  ```bash
  helm template release ./chart -f values.yaml | kubectl apply -f - --dry-run=server
  ```

Chart Authoring: structure & key files
- Typical chart layout:
  ```
  mychart/
  ├── Chart.yaml         # metadata (name, version, apiVersion, dependencies)
  ├── values.yaml        # default values
  ├── charts/            # chart dependencies (when vendored)
  ├── templates/         # Kubernetes resource templates
  │   ├── _helpers.tpl
  │   ├── deployment.yaml
  │   └── ...
  └── README.md
  ```
- Chart.yaml important fields:
  ```yaml
  apiVersion: v2            # Helm v3 charts must be v2
  name: mychart
  description: A brief description
  type: application         # or library
  version: 0.1.0            # chart version
  appVersion: "1.2.3"       # the app image version
  dependencies:
    - name: redis
      version: 15.3.0
      repository: "https://charts.bitnami.com/bitnami"
  ```
- Library charts: `type: library` for shared template helpers.
- Values.yaml best practices:
  - Provide sensible defaults.
  - Keep secrets out of values.yaml in Git. Use external secrets or encrypted files (see Helm secrets).
  - Provide comments/documentation for values.

Chart Dependencies & Umbrella Charts
- Add dependency in Chart.yaml and maintain `Chart.lock`:
  ```yaml
  dependencies:
  - name: redis
    version: 14.8.8
    repository: "https://charts.bitnami.com/bitnami"
  ```
- Update dependencies:
  ```bash
  helm dependency update ./mychart
  helm dependency build ./mychart
  ls mychart/charts
  ```
- Umbrella chart: a parent chart that references subcharts in `charts/` or `Chart.yaml` dependencies. Use carefully to control lifecycle of subcharts.

Hooks, Tests & CRDs
- Hooks: lifecycle actions for custom resources or jobs.
  - Example hook annotation:
    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: db-migrate
      annotations:
        "helm.sh/hook": pre-upgrade,pre-install
        "helm.sh/hook-weight": "0"
        "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    ```
  - Common hooks: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `test`.
- Helm tests:
  - Add test Job with annotation `helm.sh/hook: test`.
  - Run tests:
    ```bash
    helm test my-release -n my-namespace
    ```
- CRD handling:
  - Place CRDs in `crds/` directory of chart (installed before templates).
  - CRDs are not upgraded automatically by Helm; must manage CRD upgrades carefully (CRD changes are often breaking).
  - Avoid templating CRDs — keep them static or use a dedicated CRD-install mechanism.

OCI registries & chart publishing
- Helm supports OCI registries (networks and cloud-native registries):
  ```bash
  # login to registry (example Azure Container Registry)
  helm registry login myregistry.azurecr.io --username $USER --password $PASS

  # package chart and push
  helm package ./mychart
  helm push mychart-0.1.0.tgz oci://myregistry.azurecr.io/helm

  # directly push dir (requires helm push plugin or helm cli support)
  helm chart save ./mychart oci://registry.example.com/myrepo/mychart:0.1.0
  helm chart push oci://registry.example.com/myrepo/mychart:0.1.0

  # pull
  helm chart pull oci://registry.example.com/myrepo/mychart:0.1.0
  helm chart export oci://registry.example.com/myrepo/mychart:0.1.0 --destination ./charts
  ```
  Verification:
  ```bash
  # list published versions
  # via registry UI or `helm chart list` (depends on version)
  ```
- ChartMuseum and other servers: publish via `curl` or `chart-releaser`.

Signing, provenance & chart security
- Sign chart packages (requires GPG key):
  ```bash
  helm package ./mychart --sign --key "my-key" --keyring /path/to/pubring.kbx
  ```
  - Verify signatures with the consuming tool or `helm verify` if available (plugin/tooling dependent).
- Verify chart provenance via `prov` files (older tools) — ensure consumers verify integrity.
- Use CI to run `helm lint`, `chart-testing` and `kubeconform`/`conftest` to validate manifests.

Plugins & recommended tooling
- Useful plugins:
  - helm-diff: `helm plugin install https://github.com/databus23/helm-diff` — shows diffs between releases/upgrade
  - helm-namespace, helm-secrets (integration with SOPS), helm-push, chart-releaser
- List plugins:
  ```bash
  helm plugin list
  ```
- Example using helm-diff before upgrade:
  ```bash
  helm repo update
  helm diff upgrade my-app bitnami/nginx -f values-prod.yaml
  ```

CI/CD patterns (example: GitHub Actions)
- Basic pipeline flow:
  - Lint chart (`helm lint`)
  - Render & validate manifests (`helm template` + `kubeval`/`kubeconform`)
  - Run chart-testing and unit tests
  - Package chart & publish to repo or OCI
  - Deploy with `helm upgrade --install --atomic --wait`
- GitHub Actions sample (simplified):
  ```yaml
  name: CI
  on: [push]
  jobs:
    lint-and-test:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v4
      - name: Set up Helm
        uses: azure/setup-helm@v3
      - name: Lint
        run: helm lint ./chart
      - name: Template & validate
        run: helm template test ./chart -f values-ci.yaml | kubeconform -strict
  ```
- GitOps (Argo CD / Flux):
  - Store rendered or raw chart (Chart + values) in Git repo.
  - Argo/Flux will sync the release to the cluster (Argo's `HelmRelease` CRD or Flux v2 HelmController).
  - Keep chart version pinned for reproducibility.

Helmfile / Chartfile workflows
- Helmfile centralizes multiple releases and environment values.
  - Example `helmfile.yaml`:
    ```yaml
    repositories:
      - name: bitnami
        url: https://charts.bitnami.com/bitnami

    releases:
      - name: redis
        namespace: data
        chart: bitnami/redis
        version: 15.3.0
        values:
          - values/redis-prod.yaml

      - name: web
        namespace: web
        chart: ./charts/web
        values:
          - values/web-prod.yaml
    ```
  - Commands:
    ```bash
    helmfile repos
    helmfile sync             # install/upgrade everything
    helmfile apply            # sync + apply
    helmfile destroy          # uninstall
    ```
  - Useful for multi-chart orchestration.

Best practices & production checklist
- Use immutable artifact references: pin chart versions & image tags.
- Keep values.yaml for non-sensitive defaults; secrets stored elsewhere (Vault, SealedSecrets, SOPS).
- Use `--atomic` and `--wait` during production upgrades; set reasonable `--timeout`.
- Test chart changes in staging (helm test) before production.
- Keep CRDs separate and manage upgrades carefully.
- Use `helm diff` or `helm template` in CI to prevent surprises.
- Use PodDisruptionBudgets, readiness/liveness probes, resource requests & limits.
- Use `chart-testing` and `kubeconform` or `conftest` for validating rendered manifests.
- Run `helm lint` and `yamllint` in CI.

Troubleshooting common issues & commands
- Release stuck in PENDING_UPGRADE / PENDING_INSTALL:
  - Inspect history and status:
    ```bash
    helm history <release> -n <ns>
    helm status <release> -n <ns> --debug
    ```
  - Check Kubernetes events and pods:
    ```bash
    kubectl get events -n <ns> --sort-by='.metadata.creationTimestamp'
    kubectl describe pod -n <ns> <pod>
    kubectl logs -n <ns> <pod>
    ```
  - If release failed and left resources: consider `helm rollback` or `helm uninstall --keep-history` then reinstall.
- Upgrade shows "cannot patch" / immutable field errors:
  - Some Kubernetes fields are immutable (e.g., Service.clusterIP). You may need to:
    - Delete only that resource (if safe) and let Helm recreate it (risk downtime).
    - Use `helm upgrade --force` (deletes and recreates resources) — use with caution.
- ImagePullBackOff or CrashLoopBackOff after helm upgrade:
  - Inspect rendered image tag:
    ```bash
    helm get manifest my-release -n ns | grep image:
    kubectl describe pod <pod> -n ns
    kubectl logs <pod> -n ns --previous
    ```
  - Ensure image registry credentials (`imagePullSecrets`) are set.
- Hooks failing or not running:
  - Inspect release hooks:
    ```bash
    helm get hooks my-release -n ns
    ```
  - Use `helm get all` to see hook resources and their status.
- CRD conflicts:
  - CRDs managed by separate operator or managed manually; Helm won't upgrade CRDs in `crds/`. Use a separate process for CRD migrations.
- Secrets become visible in helm get manifest:
  - Avoid storing secrets in values.yaml. Use external secret managers.
- Verify template errors:
  - Use `helm template --debug --set key=value` to spot template function errors (e.g., nil pointer, missing keys).

Appendix: Quick command cheat sheet
- Install:
  ```bash
  helm install RELEASE CHART -n NAMESPACE --create-namespace -f values.yaml --set key=val
  ```
- Upgrade (install if missing):
  ```bash
  helm upgrade --install RELEASE CHART -n NAMESPACE -f values.yaml --atomic --wait --timeout 10m
  ```
- Uninstall:
  ```bash
  helm uninstall RELEASE -n NAMESPACE
  ```
- Inspect:
  ```bash
  helm list -n NAMESPACE
  helm status RELEASE -n NAMESPACE
  helm history RELEASE -n NAMESPACE
  helm get values RELEASE -n NAMESPACE
  helm get manifest RELEASE -n NAMESPACE
  helm get hooks RELEASE -n NAMESPACE
  ```
- Template & lint:
  ```bash
  helm template RELEASE CHART -f values.yaml
  helm lint ./chart
  helm diff upgrade RELEASE CHART -f values.yaml    # plugin: helm-diff
  ```
- Package & publish:
  ```bash
  helm package ./chart
  helm push chart.tgz repo-name                # plugin or chart-releaser
  helm chart save ./chart oci://registry/repo/mychart:1.0.0
  helm chart push oci://registry/repo/mychart:1.0.0
  ```
- Work with dependencies:
  ```bash
  helm dependency update ./chart
  helm dependency build ./chart
  ```

Further reading & tools
- Official Helm docs: https://helm.sh/docs/
- Chart Best Practices: https://helm.sh/docs/chart_best_practices/
- Useful tools: helm-diff, helm-secrets (SOPS), chart-releaser, kubeconform, conftest, chart-testing, kubeval
- GitOps: ArgoCD (Helm support), Flux CD (Helm Controller)

- A GitHub Actions workflow that packages, signs, and pushes charts to OCI registry and deploys to a cluster with `helm upgrade --install`.

Which of the above should I produce next (single file, sample repo layout, GitHub Action, or push to your repo)?  
