# Hello World on EKS — Full Deployment Guide

End-to-end GitOps deployment of a Hello World app on Amazon EKS with ArgoCD, Istio mTLS, External Secrets Operator, and Crossplane.

---

## Repositories

| # | Repository | Purpose |
|---|---|---|
| 1 | `https://github.com/Shlomi-Lantser/eks-terraform` | Terraform IaC — EKS, VPC, ECR, IAM roles, ESO, Crossplane |
| 2 | `https://github.com/Shlomi-Lantser/hello-world-deployments` | GitOps repo — Helm chart managed by ArgoCD |
| 3 | `https://github.com/Shlomi-Lantser/hello-world-app` | Application source code, Dockerfile, CI/CD pipeline |
| 4 | `https://github.com/Shlomi-Lantser/hello-world-manifests` | Kubernetes manifests — ESO, Crossplane, ArgoCD App + all Helm values |

> CI/CD config lives inside `hello-world-app` at `.github/workflows/ci.yaml`

---

## Architecture

### Traffic Flow

```
Internet
   │
   ▼
AWS NLB  (provisioned by AWS Load Balancer Controller)
   │  port 80  → HTTP
   │  port 443 → HTTPS (TLS terminated by Istio, self-signed cert from Secrets Manager)
   ▼
Istio IngressGateway  (Envoy proxy — istio-system)
   │  Gateway + VirtualService
   ▼
hello-world Pod  (2/2 — app + istio-proxy sidecar)
   │  mTLS STRICT enforced by PeerAuthentication
   ▼
FastAPI  (port 9000) — GET /  GET /hello  GET /health
```

### CI/CD Flow

```
git push → hello-world-app (main branch)
   │
   ▼
GitHub Actions
   ├── Build Docker image
   ├── Push to ECR with SHA tag + latest
   └── Update charts/hello-world/values.yaml in hello-world-deployments
       via SSH deploy key → commit + push
   │
   ▼
ArgoCD detects Git change → Helm chart sync → rolling deploy on EKS
```

### Secrets Flow

```
AWS Secrets Manager
   ├── hello-world/dev/argocd-admin-password  →  ESO  →  argocd-secret (argocd ns)
   └── hello-world/dev/istio-tls              →  ESO  →  hello-world-tls (istio-system ns)
```

---

## Prerequisites

### Tools

- Terraform >= 1.11.0
- AWS CLI v2
- kubectl
- Helm >= 3.0
- openssl
- git

### AWS Authentication

Authentication uses an **AWS access key and secret** — no SSO required:

```bash
aws login

aws sts get-caller-identity
```

---

## Repository Layout

### 1. eks-terraform

```
eks-terraform/
├── versions.tf                  # Provider versions + S3 backend config
├── vpc.tf                       # VPC, subnets, NAT gateway, subnet tags for LBC
├── eks.tf                       # EKS cluster, managed node group, addons
├── irsa.tf                      # AWS LBC — Pod Identity association + Helm release
├── ecr.tf                       # ECR repo + GitHub Actions OIDC IAM role
├── eso.tf                       # ESO IAM role + ArgoCD password in Secrets Manager
│                                # Terraform bcrypt()s the password before storing
├── crossplane.tf                # Crossplane S3 IAM role — Pod Identity
├── variables.tf
├── outputs.tf
└── terraform.tfvars            

```

### 2. hello-world-deployments

```
hello-world-deployments/
└── charts/
    └── hello-world/
        ├── Chart.yaml
        ├── values.yaml                  # image.tag always quoted string
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml             # disabled — traffic via Istio Gateway
            ├── gateway.yaml             # HTTP port 80 + HTTPS port 443
            ├── virtual-service.yaml
            └── peer-authentication.yaml # mTLS STRICT
```

### 3. hello-world-app

```
hello-world-app/
├── app.py               # FastAPI — GET /, /hello, /health on port 9000
├── Dockerfile           # python:3.12-slim, non-root user
├── requirements.txt
├── README.md
└── .github/
    └── workflows/
        └── ci.yaml      # Build → ECR push → update deployments repo tag via SSH
```

### 4. hello-world-manifests

```
hello-world-manifests/
├── argo/
│   └── argocd-app.yaml
├── eso/
│   ├── cluster-secret-store.yaml
│   ├── argocd-external-secret.yaml      # syncs argocd-secret (bcrypt hash + secretkey)
│   └── istio-tls-external-secret.yaml   # syncs hello-world-tls (cert + key) to istio-system
├── crossplane/
│   ├── runtime-config.yaml              # serviceAccountTemplate — creates the SA
│   ├── provider.yaml
│   ├── provider-config.yaml             # source: WebIdentity (Pod Identity)
│   ├── s3-bucket.yaml
│   └── s3-bucket-versioning.yaml
├── hello-world-app/
│   └── namespace.yaml                   # hello-world namespace with istio-injection label
└── helm-values/
    ├── argo/
    │   └── argocd-values.yaml           # configs.secret.createSecret: false
    ├── alb-controller/
    │   └── aws-load-balancer-controller-values.yaml
    ├── external-secrets/
    │   └── external-secrets-values.yaml
    ├── crossplane/
    │   └── crossplane-values.yaml
    └── istio/
        ├── istiod-values.yaml
        └── istio-gateway-values.yaml    # NLB healthcheck on port 15021
```

---

## Deployment — Step by Step

### Step 1 — Clone All Repositories

```bash
git clone https://github.com/Shlomi-Lantser/eks-terraform
git clone https://github.com/Shlomi-Lantser/hello-world-app
git clone https://github.com/Shlomi-Lantser/hello-world-deployments
git clone https://github.com/Shlomi-Lantser/hello-world-manifests
```

---

### Step 2 — Create Terraform Backend (s3 bucket)

Create the S3 bucket used to store Terraform remote state.


Then fill in the `backend "s3"` block in `versions.tf`:

```hcl
  backend "s3" {
    bucket         = "your-bucket"
    key            = "eks-hello-world/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    use_lockfile = true
    profile = "assignment-profile"
  }
 }
```

---

### Step 3 — Configure terraform.tfvars

---

### Step 4 — Terraform Apply

Provisions EKS cluster, VPC, ECR repository, IAM roles for LBC/ESO/Crossplane/GitHub Actions, and stores the ArgoCD bcrypt hash in AWS Secrets Manager.

```bash
cd eks-terraform
terraform init
terraform plan
terraform apply
```

Expected duration: ~15 minutes. Save the outputs:

```bash
terraform output vpc_id
terraform output ecr_repository_url
terraform output github_actions_role_arn
```

---

### Step 5 — Configure kubectl

```bash
aws eks update-kubeconfig --region eu-west-1 --name hello-world-dev-cluster

# Verify
kubectl get nodes
```

---

### Step 6 — Generate TLS Certificate and Store in Secrets Manager

A self-signed cert is used for HTTPS on the Istio Gateway. The cert and key are stored in AWS Secrets Manager — ESO syncs them into the cluster as a Kubernetes TLS secret. No cert files are committed to Git.

```bash
# Generate cert
openssl req -x509 -newkey rsa:4096 \
  -keyout tls.key \
  -out tls.crt \
  -days 730 \
  -nodes \
  -subj "/C=IL/ST=Tel Aviv/O=HelloWorld/CN=hello-world" \
  -addext "subjectAltName=DNS:hello-world,DNS:localhost"

# Verify
openssl x509 -in tls.crt -noout -subject -dates

# Store in Secrets Manager (base64 encoded)
aws secretsmanager create-secret \
  --name hello-world/dev/istio-tls \
  --region eu-west-1 \
  --secret-string "{\"tls.crt\":\"$(base64 -w0 tls.crt)\",\"tls.key\":\"$(base64 -w0 tls.key)\"}"

# Clean up local files — not needed anymore
rm tls.crt tls.key
```

> On Windows PowerShell use `[Convert]::ToBase64String([IO.File]::ReadAllBytes("tls.crt"))` instead of `base64 -w0`

---

### Step 7 — Install Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# 1. Base CRDs
helm install istio-base istio/base -n istio-system --create-namespace

# 2. Control plane
helm install istiod istio/istiod \
  -n istio-system \
  -f hello-world-manifests/helm-values/istio/istiod-values.yaml \
  --wait

# 3. Ingress Gateway — triggers NLB creation once LBC is running
helm install istio-ingressgateway istio/gateway \
  -n istio-system \
  -f hello-world-manifests/helm-values/istio/istio-gateway-values.yaml

kubectl get pods -n istio-system
```

---

### Step 8 — Install AWS Load Balancer Controller

The LBC provisions the NLB for the Istio IngressGateway.
The VPC ID **must be passed explicitly** — LBC cannot detect it from instance metadata on EKS v21+.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

VPC_ID=$(cd eks-terraform && terraform output -raw vpc_id)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f hello-world-manifests/helm-values/alb-controller/aws-load-balancer-controller-values.yaml \
  --set vpcId=$VPC_ID

# Verify LBC is Running
kubectl get pods -n kube-system | grep aws-load-balancer

# Watch NLB provisioning (~2 minutes)
kubectl get svc istio-ingressgateway -n istio-system -w
# Wait until EXTERNAL-IP is populated with the NLB DNS name
```

---

### Step 9 — Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  -f hello-world-manifests/helm-values/external-secrets/external-secrets-values.yaml

kubectl get pods -n external-secrets
```

---

### Step 10 — Create argocd Namespace and Apply ESO Manifests

> **Critical ordering:**
> 1. The `argocd` namespace must exist before the ExternalSecret is applied.
> 2. `argocd-secret` must be fully populated by ESO **before** ArgoCD installs.
>
> This prevents ArgoCD from auto-generating its own password on startup.

```bash
# Create namespace first
kubectl create namespace argocd

# Connect ESO to AWS Secrets Manager
kubectl apply -f hello-world-manifests/eso/cluster-secret-store.yaml

# Pre-create argocd-secret with bcrypt hash + server.secretkey
kubectl apply -f hello-world-manifests/eso/argocd-external-secret.yaml

# Wait for sync (~5-10 seconds)
kubectl get externalsecret argocd-admin-password -n argocd
# STATUS must show: SecretSynced

# Verify argocd-secret has the bcrypt hash
kubectl get secret argocd-secret -n argocd \
  -o jsonpath='{.data.admin\.password}' | base64 -d
# Must return a hash starting with $2b$ or $2a$
```

---

### Step 11 — Sync Istio TLS Secret from Secrets Manager

ESO syncs the TLS cert and key stored in Secrets Manager into `istio-system` as a Kubernetes TLS secret. This must exist before ArgoCD deploys the Helm chart.

```bash
kubectl apply -f hello-world-manifests/eso/istio-tls-external-secret.yaml

# Wait for sync
kubectl get externalsecret istio-tls -n istio-system -w
# STATUS must show: SecretSynced

# Verify secret was created
kubectl get secret hello-world-tls -n istio-system
```

---

### Step 12 — Install ArgoCD

ArgoCD finds `argocd-secret` already populated by ESO and does not generate its own password.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  -n argocd \
  -f hello-world-manifests/helm-values/argo/argocd-values.yaml

kubectl get pods -n argocd -w

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open:     https://localhost:8080
# Username: admin
# Password: <plain text password from terraform.tfvars>
```

---

### Step 13 — Set Up SSH Deploy Key for CI/CD

The CI pipeline pushes image tag updates to `hello-world-deployments` using an SSH deploy key.

```bash
# Generate key pair (no passphrase)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f deploy_key -N ""
```

**Add public key to `hello-world-deployments`:**
- GitHub → `hello-world-deployments` → Settings → Deploy keys → Add deploy key
- Title: `github-actions`
- Key: contents of `deploy_key.pub`
- ✅ Allow write access

**Add secrets to `hello-world-app`:**
- GitHub → `hello-world-app` → Settings → Secrets and variables → Actions

| Secret | Value |
|---|---|
| `AWS_ACCOUNT_ID` | your AWS account ID |
| `AWS_REGION` | `eu-west-1` |
| `DEPLOYMENTS_SSH_KEY` | contents of `deploy_key` (private key) |

```bash
# Clean up local key files
rm deploy_key deploy_key.pub
```

---

### Step 14 — Trigger CI/CD Pipeline

The Docker image must exist in ECR before ArgoCD can deploy the app.

```bash
cd hello-world-app
git commit --allow-empty -m "ci: trigger initial build"
git push origin main
```

The pipeline will:
1. Build the Docker image
2. Push to ECR with SHA tag (quoted string — prevents YAML scientific notation) and `latest`
3. Clone `hello-world-deployments` via SSH deploy key
4. Update `charts/hello-world/values.yaml`: `tag: "abc1234"`
5. Commit and push

Verify the image exists:

```bash
aws ecr describe-images \
  --repository-name hello-world \
  --region eu-west-1 \
  --query 'imageDetails[*].imageTags'
```

---

### Step 15 — Deploy App via ArgoCD

```bash
# Apply namespace manifest (includes istio-injection label)
kubectl apply -f hello-world-manifests/hello-world-app/namespace.yaml

# Register ArgoCD Application
kubectl apply -f hello-world-manifests/argo/argocd-app.yaml

# Watch sync
kubectl get application hello-world -n argocd -w
# Wait for: Synced / Healthy
```

---

### Step 16 — Verify

```bash
# Pods must show 2/2 (app + istio-proxy sidecar)
kubectl get pods -n hello-world

# Confirm sidecar
kubectl get pod -n hello-world -l app=hello-world \
  -o jsonpath='{.items[0].spec.containers[*].name}'
# Expected: hello-world istio-proxy

# mTLS
kubectl get peerauthentication -n hello-world
# Expected: STRICT

# Get public URL
NLB=$(kubectl get svc istio-ingressgateway -n istio-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Public URL: http://$NLB"

# Test HTTP
curl http://$NLB/hello
# Test HTTPS (self-signed cert)
curl -k https://$NLB/hello
# Expected: {"message": "Hello World"}

# Validate Istio is serving the correct cert
istioctl proxy-config secret -n istio-system \
  $(kubectl get pod -n istio-system -l app=istio-ingressgateway -o jsonpath='{.items[0].metadata.name}')
```

---

## Bonus Implementations

### Bonus 1 — Crossplane (S3 Bucket)

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  -n crossplane-system --create-namespace \
  -f hello-world-manifests/helm-values/crossplane/crossplane-values.yaml

kubectl get pods -n crossplane-system -w

# Apply in strict order — RuntimeConfig MUST come before Provider
kubectl apply -f hello-world-manifests/crossplane/runtime-config.yaml

kubectl apply -f hello-world-manifests/crossplane/provider.yaml
kubectl get provider provider-aws-s3 -w
# Wait for: HEALTHY=True (~2 minutes)

kubectl apply -f hello-world-manifests/crossplane/provider-config.yaml
kubectl apply -f hello-world-manifests/crossplane/s3-bucket.yaml
kubectl apply -f hello-world-manifests/crossplane/s3-bucket-versioning.yaml
```

**Verify:**
```bash
kubectl get bucket
# NAME                          READY   SYNCED
# hello-world-crossplane-demo   True    True
```
AWS Console: S3 → Buckets → `hello-world-crossplane-demo` → Tags: `ManagedBy=crossplane`

---

### Bonus 2 — Istio + mTLS + HTTPS

Already deployed in Steps 7 and 11.

```bash
# Sidecar injection — pods show 2/2
kubectl get pods -n hello-world

# mTLS STRICT
kubectl get peerauthentication -n hello-world

# HTTPS with cert from Secrets Manager
curl -k https://$NLB/hello

# Confirm cert subject matches what was generated
curl.exe -kv https://$NLB/hello 2>&1 | grep "subject\|issuer"
# Expected: CN=hello-world, O=HelloWorld
```

---

### Bonus 3 — External Secrets Operator + Secrets Manager

Already deployed in Steps 9–11. ESO manages two secrets:

```bash
# ArgoCD password
kubectl get externalsecret argocd-admin-password -n argocd
# STATUS: SecretSynced

# Istio TLS cert + key
kubectl get externalsecret istio-tls -n istio-system
# STATUS: SecretSynced

# ClusterSecretStore
kubectl get clustersecretstore
# STATUS: Valid
```

**What ESO manages:**
- `hello-world/dev/argocd-admin-password` → `argocd-secret` in `argocd` namespace (bcrypt hash + server.secretkey)
- `hello-world/dev/istio-tls` → `hello-world-tls` in `istio-system` namespace (TLS cert + key for HTTPS)

---

## Run Locally

```bash
cd hello-world-app
pip install -r requirements.txt
python app.py

curl http://localhost:9000/
curl http://localhost:9000/hello
curl http://localhost:9000/health
```

**With Docker:**

```bash
docker build -t hello-world .
docker run -p 9000:9000 hello-world
curl http://localhost:9000/hello
```

---

## Design Decisions

**Pod Identity over IRSA** — All pod-level AWS authentication uses EKS Pod Identity. No OIDC provider config, no ServiceAccount annotations. AWS rotates credentials automatically.

**Terraform bcrypt() for password management** — The ArgoCD password is set as plain text in `terraform.tfvars`. Terraform hashes it with `bcrypt()` before writing to Secrets Manager. ESO syncs the hash into the cluster. One plain text secret to manage — everything else is automated.

**TLS cert stored in Secrets Manager** — The self-signed TLS cert and key are stored in AWS Secrets Manager and synced into `istio-system` via ESO. No cert files committed to Git. Cert rotation is a single Secrets Manager update — ESO re-syncs automatically within the `refreshInterval`.

**NLB healthcheck on port 15021** — The NLB must target port `15021` (`/healthz/ready`) not the traffic port. Without this annotation the NLB intermittently marks targets unhealthy causing `ERR_CONNECTION_REFUSED`.

**HTTPS with self-signed cert** — AWS ACM cannot issue certificates for `*.elb.amazonaws.com` hostnames. A custom domain would be required for a CA-signed cert. HTTPS is fully functional end-to-end via Istio TLS termination — `curl -k` skips browser validation of the self-signed cert.

**ESO must run before ArgoCD** — `argocd-secret` must be pre-created by ESO before ArgoCD installs. `configs.secret.createSecret: false` in ArgoCD Helm values prevents auto-generation. The ExternalSecret uses `creationPolicy: Owner` and a template to set all three required fields in a single sync.

**Image tag must be a quoted string** — A 7-character SHA that is all digits (e.g. `5551977`) is interpreted by YAML as a float (`5.551977e+06`), causing `InvalidImageName`. The CI pipeline wraps the tag in quotes: `tag: "5551977"`.

**Crossplane serviceAccountTemplate** — `DeploymentRuntimeConfig` must use `serviceAccountTemplate.metadata.name`. This tells Crossplane to **create** the SA. Using `deploymentTemplate.spec.serviceAccountName` only references an existing SA, causing the provider pod to fail with `serviceaccount not found`.

---
