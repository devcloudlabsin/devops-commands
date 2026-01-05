# Kubernetes tools for Windows VDI — EKS & Helm deployments via Jenkins and GitLab
Date: 2026-01-05  
Author: devcloudlabsin

Purpose
- A step-by-step, enterprise-ready guide focused on installing and configuring Kubernetes tooling on Windows VDI clients for working with Amazon EKS and deploying Helm charts via Jenkins and GitLab CI/CD.
- Target audience: platform engineers, DevOps, and SRE teams in large enterprise banking environments where security, auditability, and compliance are required.
- Covers Windows VDI setup (with WSL2 option), secure access patterns to EKS, tooling installation, best practices, detailed commands and examples for Jenkins pipelines and GitLab CI pipelines, and enterprise security considerations.

Contents
- 1. Enterprise context & requirements
- 2. Windows VDI preparation (policies, proxies, privileges)
- 3. Installation of tools (PowerShell, Chocolatey/winget, WSL2)
  - 3.x Additional kubectl ecosystem tools (krew, kubectx/kubens, kustomize, kind, k9s, stern, kubeseal, kubeval, kube-score, kube-ps1)
- 4. Configure AWS & EKS access (IAM, SSO, kubeconfig, RBAC)
- 5. Helm basics & secure usage
- 6. Jenkins pipeline example (build, scan, push, deploy with Helm)
- 7. GitLab CI pipeline example (build, scan, push, deploy)
- 8. EKS production recommendations for banking (networking, security, audit)
- 9. Secrets management and encryption
- 10. Monitoring, logging and policy checks
- 11. Troubleshooting & FAQs
- 12. Appendices (sample files & commands)

---

1. Enterprise context & requirements
- Banking requirements typically include:
  - Least-privilege access; actions are auditable (CloudTrail, Control Plane logs).
  - Private EKS clusters or private control-plane endpoints; no open internet access from nodes.
  - Centralized secrets management (Vault or AWS Secrets Manager + SOPS).
  - Image scanning (Trivy/Clair) and signed images (cosign).
  - Approvals for production deploys, change management, and CI audit trail.
  - Network segmentation: separate VPCs, subnets, security groups; egress control.
  - Monitoring/alerting and retention policies.
- This guide gives commands and pipelines that conform to these constraints where possible.

---

2. Windows VDI preparation
2.1. VDI baseline policies
- Work with desktop engineering to ensure:
  - VDI image supports WSL2 (recommended) or provides Docker Desktop enterprise with WSL2 backend.
  - Chocolatey or winget is allowed for tool installation, or provide a signed MSI/EXE.
  - Required ports outbound to AWS endpoints (EKS, ECR, S3, STS) are allowed through proxy/firewall.
  - SSO / ADFS / Azure AD is configured for enterprise authentication if required.

2.2. Proxy and environment variables
- If corporate proxy is present, configure:
  - Windows environment variables (for PowerShell sessions):
```powershell
setx HTTP_PROXY "http://proxy.company.local:8080"
setx HTTPS_PROXY "http://proxy.company.local:8080"
setx NO_PROXY "169.254.169.254,localhost,127.0.0.1,.company.local"
```
- For WSL2, add to `/etc/environment` or shell profile:
```bash
export HTTP_PROXY="http://proxy.company.local:8080"
export HTTPS_PROXY="http://proxy.company.local:8080"
export NO_PROXY="169.254.169.254,localhost,127.0.0.1,.company.local"
```

2.3. Admin rights & signed installers
- Some installers require elevated rights. Use signed binaries approved by desktop security.
- Use internal artifact repositories for tools if corporate policy forbids external downloads.

---

3. Installation of tools (Windows + WSL2 recommended)
Recommendation: use WSL2 (Ubuntu) on Windows VDI for parity with Linux CI environments. Alternatively use native Windows tools.

3.1. Install WSL2 (if allowed)
PowerShell (Admin):
```powershell
wsl --install -d ubuntu
# restart when prompted
```
Open Ubuntu shell and update:
```bash
sudo apt update && sudo apt upgrade -y
```

3.2. Install core tools (recommended list)
- aws-cli v2
- eksctl
- kubectl
- helm (v3)
- docker (or Docker Desktop with WSL2)
- jq, git, make, openssh
- trivy (image scanner), cosign (image signing), sops (secret encryption)
- kubelogin (for OIDC SSO if needed)
- kind (for local clusters)
- kustomize (manifests customization)
- krew (kubectl plugin manager) + useful plugins
- kubectx / kubens (context & namespace switching)
- k9s (terminal UI)
- stern or kubetail (tailing logs)
- kubeval, kube-score (manifest validation)
- kubeseal (sealed secrets)

3.3. Example installs (WSL2 / Ubuntu)
```bash
# dependencies
sudo apt-get install -y curl unzip git apt-transport-https ca-certificates gnupg lsb-release jq

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# kubectl (stable)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz
sudo mv eksctl /usr/local/bin

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# docker (if not using Docker Desktop)
sudo apt-get install -y docker.io
sudo usermod -aG docker $(whoami)

# kind (Kubernetes in Docker) - adjust version if needed
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.20.0/kind-$(uname)-amd64"
chmod +x ./kind && sudo mv kind /usr/local/bin

# kustomize (plugin or standalone)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin

# trivy
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_$(uname -s)_amd64.deb
sudo dpkg -i trivy_*.deb

# cosign
COSIGN_VER=$(curl -s https://api.github.com/repos/sigstore/cosign/releases/latest | jq -r .tag_name)
curl -Lo cosign "https://github.com/sigstore/cosign/releases/download/${COSIGN_VER}/cosign-${COSIGN_VER}-linux-amd64"
chmod +x cosign && sudo mv cosign /usr/local/bin/
```

3.4. Windows-native installations (PowerShell & Chocolatey/winget)
```powershell
# If Chocolatey available (run as admin)
choco install -y awscli kubernetes-cli helm docker-desktop kind kustomize k9s kubectx kubens

# or winget (requires internet & policies)
winget install --id Amazon.AWSCLI
winget install --id Kubernetes.kubectl
winget install --id Helm.Helm
winget install --id Docker.DockerDesktop
```

3.5. Verify installations
```bash
aws --version
eksctl version
kubectl version --client
helm version
docker --version
kind --version
kustomize version
trivy --version
cosign version
```

3.6. Additional kubectl ecosystem tools (detailed, recommended for Windows VDI)
This section covers tools that enhance kubectl workflows: context/namespace switching, plugin management (krew), manifest customization (kustomize), local clusters (kind), observability tools (k9s, stern), policy/validation (kubeval, kube-score), and secrets tooling (kubeseal, sops). Each entry includes installation and common usage patterns for WSL2 and Windows.

3.6.1. krew — kubectl plugin manager
Why: krew lets you discover and install kubectl plugins using a unified interface. Use it to add productivity plugins (ctx/ns, neat, view-secret, who-can, sniff, etc).

Install (WSL2 / Linux):
```bash
(
  set -x
  cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m)" &&
  if [ "$ARCH" = "x86_64" ]; then ARCH="amd64"; fi &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-${OS}_${ARCH}.tar.gz" &&
  tar zxvf krew-*.tar.gz &&
  ./krew-"${OS}_${ARCH}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
# put the export line into ~/.bashrc or ~/.profile
```

Install (Windows PowerShell):
```powershell
# Download krew release for windows_amd64 and unzip
Invoke-WebRequest -UseBasicParsing -Uri "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-windows_amd64.zip" -OutFile "$env:TEMP\krew.zip"
Expand-Archive -Path "$env:TEMP\krew.zip" -DestinationPath "$env:TEMP\krew"
# run installer
& "$env:TEMP\krew\krew-windows_amd64.exe" install krew
# Add to PATH: $env:USERPROFILE\.krew\bin
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";" + "$env:USERPROFILE\.krew\bin", "User")
```

Common krew usage:
```bash
kubectl krew update
kubectl krew install ctx ns neat view-secret who-can sniff
# list installed plugins
kubectl krew list
```

3.6.2. kubectx & kubens (context & namespace switching)
Why: fast switching between kubeconfig contexts and namespaces.

Install options:
- via krew (preferred to keep everything under kubectl/krew):
  - If krew installed, use `kubectl krew install ctx` and `kubectl krew install ns` (plugins names may be `ctx`/`ns` or `kubectx`/`kubens` depending on krew index; check `kubectl krew search ctx`).
- via Chocolatey on Windows:
  - `choco install kubectx`
- manual:
```bash
git clone https://github.com/ahmetb/kubectx.git ~/.kubectx
# add ~/.kubectx to PATH or symlink scripts to /usr/local/bin
```

Usage:
```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context my-eks-cluster

# kubectx (alias)
kubectx my-eks-cluster
# kubens (alias)
kubens my-namespace
# If installed via krew use: kubectl ctx <context> and kubectl ns <namespace>
```

Tip: Add short aliases in your shell:
```bash
alias k='kubectl'
alias kc='kubectl ctx'
alias kn='kubectl ns'
```

3.6.3. kustomize — declarative manifest customization
Why: overlay configuration for environments without templating languages. Supports patches, overlays and generators.

Install:
- Many kubectl versions include `kubectl kustomize` or `kubectl apply -k`, but standalone kustomize has features and releases.
- WSL2 install (standalone):
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```
- Windows (choco): `choco install kustomize`

Usage:
```bash
kustomize build overlays/prod | kubectl apply -f -
# or with kubectl
kubectl apply -k overlays/prod
```

3.6.4. kind — Kubernetes in Docker (local clusters)
Why: Lightweight local clusters for testing manifests and CI functional tests.

Install (WSL2):
```bash
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.20.0/kind-$(uname)-amd64"
chmod +x ./kind && sudo mv kind /usr/local/bin
```
Windows (choco): `choco install kind`

Create cluster:
```bash
kind create cluster --name dev --config kind-config.yaml
# use kubeconfig automatically added to ~/.kube/config
kubectl cluster-info --context kind-dev
```

3.6.5. k9s — terminal UI for Kubernetes
Why: interactive terminal to browse resources quickly.

Install:
- WSL2: `curl -sS https://webinstall.dev/k9s | bash` or download release binary.
- Windows: `choco install k9s`

Run:
```bash
k9s
```

3.6.6. stern & kubetail — multi-pod log tailing
Why: tail logs from multiple pods/containers with coloring and filters.

Install:
- stern: download from releases or `go install github.com/stern/stern@latest`
- kubetail: a Bash script you can curl to /usr/local/bin/kubetail

Usage:
```bash
stern deployment/myapp -n prod
# or
kubetail myapp -n prod
```

3.6.7. kubeseal (Sealed Secrets)
Why: create encrypted k8s secrets that can safely be committed to git; SealedSecrets controller in-cluster decrypts them.

Install:
```bash
# kubeseal client
curl -Lo kubeseal https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/kubeseal-linux-amd64
chmod +x kubeseal && sudo mv kubeseal /usr/local/bin
# obtain controller public key (from cluster)
kubeseal --fetch-cert > pub-cert.pem
# create sealed secret
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password=s3cr3t -o yaml --dry-run=client | kubeseal --cert pub-cert.pem -o yaml > sealedsecret.yaml
```

3.6.8. kubeval & kube-score — manifest validation
Why: validate manifests against Kubernetes schemas and best-practice checks.

Install (kubeval):
```bash
# Linux
wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
tar xzvf kubeval-*.tar.gz
sudo mv kubeval /usr/local/bin
```
Install (kube-score):
```bash
# Linux
curl -sLo kube-score.tar.gz https://github.com/zegl/kube-score/releases/latest/download/kube-score_$(uname)_amd64.tar.gz
tar xzvf kube-score.tar.gz
sudo mv kube-score /usr/local/bin
```

Usage:
```bash
kubeval ./manifests/deployment.yaml
kube-score score ./manifests/deployment.yaml
```

3.6.9. kube-ps1 — prompt showing Kubernetes context (useful in shells)
Why: quick visible context/namespace in your shell prompt to reduce accidental operations.

Install:
```bash
# add to powerline or bash prompt
git clone https://github.com/jonmosco/kube-ps1.git ~/.kube-ps1
# then source in ~/.bashrc or PowerShell profile

