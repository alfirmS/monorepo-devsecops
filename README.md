<div align="center">

# 🚀 monorepo-devsecops

**One repo to rule all deployments.**

A centralized GitOps monorepo for managing Helm charts, Kubernetes configurations, and CI/CD pipelines — auto-generated and maintained by Jenkins Service Generator.

[![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![Helm](https://img.shields.io/badge/Packaging-Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Vault](https://img.shields.io/badge/Secrets-Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=black)](https://www.vaultproject.io/)

</div>

---

## 📖 Overview

This monorepo serves as the **single source of truth** for all service deployments across multiple environments. Rather than scattered configs across dozens of repos, everything lives here — Helm charts, values, and Jenkinsfiles — structured and versioned together.

Helm charts and pipeline configs are **auto-generated** by the Jenkins Service Generator pipeline, which scaffolds everything from a starter template and commits directly to this repo.

```
monorepo-devsecops
├── templates/                    # Shared pipeline & config templates
│   └── Jenkinsfile.template      # Base Jenkinsfile for all services
│
└── services/                     # One folder per service
    └── <service-name>/
        ├── Jenkinsfile           # Auto-generated deploy pipeline
        └── helm-<service-name>/  # Helm chart (copied from starter)
            ├── Chart.yaml
            ├── values.yaml       # Patched for target environment
            └── templates/
                ├── deployment.yaml
                ├── service.yaml
                ├── hpa.yaml
                ├── pdb.yaml
                ├── ingress.yaml
                ├── externalsecrets.yaml
                └── serviceaccount.yaml
```

---

## 🌿 Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Production-ready configs, templates, and all generated service files |
| `helm-starter` | Base Helm chart template — source of truth for new service scaffolding |

---

## ⚙️ How It Works

```
┌─────────────────────────────────────────────────┐
│           Jenkins Service Generator              │
│                                                  │
│  1. Validate parameters                          │
│  2. Clone this repo (main branch)                │
│  3. Clone starter helm (helm-starter branch)     │
│  4. Copy starter → services/<name>/helm-<name>/  │
│  5. Patch values.yaml with ENV config            │
│  6. Generate Jenkinsfile from template           │
│  7. Commit & push to this repo                   │
│  8. Create Jenkins deploy job via API            │
└─────────────────────────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────┐
        │   Jenkins Deploy Job      │
        │                           │
        │  1. Checkout service repo │
        │  2. Build & push image    │
        │  3. Clone this repo       │
        │  4. helm dep update       │
        │  5. helm lint + dry-run   │
        │  6. helm upgrade --atomic │
        │  7. Post-deploy check     │
        └───────────────────────────┘
```

---

## 🏗️ Onboarding a New Service

All you need to do is run the **Service Generator** job in Jenkins with these parameters:

| Parameter | Description | Example |
|---|---|---|
| `SERVICE` | Service name (lowercase, hyphens ok) | `payment-api` |
| `ENV` | Target environment | `dev` / `uat` / `stag` / `prod` |
| `SERVICE_TYPE_K8S` | Service type | `backend` / `frontend` |
| `CONTAINER_PORT` | Exposed container port | `8080` |
| `K8S_SERVICE_TYPE` | Kubernetes service type | `ClusterIP` |
| `RESOURCE_PRESET` | Resource sizing | `small` / `medium` / `large` / `xlarge` / `custom` |
| `HPA_ENABLED` | Enable autoscaling | `true` / `false` |
| `PDB_ENABLED` | Enable pod disruption budget | `true` / `false` |
| `EXTERNAL_SECRET_ENABLED` | Enable Vault secret sync | `true` / `false` |
| `REGISTRY_TYPE` | Container registry | `Harbor` / `Huawei SWR` / `Docker Hub` |
| `REGISTRY_HOST` | Registry host URL | `registry.example.com` |
| `GIT_CRED_ID` | Jenkins credential for Git SSH | `git-credential-id` |
| `JENKINS_FOLDER` | Jenkins folder for the deploy job | `backend` |

After the generator runs, this repo will have a new commit with all files ready, and a Jenkins deploy job will be created automatically.

---

## 📦 Resource Presets

| Preset | CPU Limit | CPU Request | Memory Limit | Memory Request |
|---|---|---|---|---|
| `small` | `500m` | `250m` | `512Mi` | `256Mi` |
| `medium` | `1` | `500m` | `1Gi` | `512Mi` |
| `large` | `2` | `1` | `2Gi` | `1Gi` |
| `xlarge` | `4` | `2` | `4Gi` | `2Gi` |
| `custom` | manual | manual | manual | manual |

---

## 🌍 Environments

| ENV | Namespace | Default Branch | Kubeconfig Credential |
|---|---|---|---|
| `dev` | `development` | `dev` | `kubeconfig-dev` |
| `uat` | `uat` | `uat` | `kubeconfig-uat` |
| `stag` | `staging` | `main` | `kubeconfig-stag` |
| `prod` | `production` | `main` | `kubeconfig-prod` |

---

## 🏷️ Image Tagging

Images are tagged automatically using the format:

```
{commitId}-{ddmmyy}-{BUILD_NUMBER}

# Example:
a1b2c3d-060326-42
```

The same tag is also written to `Chart.yaml` as both `version` and `appVersion` before each deploy.

---

## 🔐 Required Jenkins Credentials

| Credential ID | Type | Used For |
|---|---|---|
| `git-credential-id` | SSH Private Key | Git clone & push |
| `harbor-registry-credential-id` | Username/Password | Harbor image push |
| `huawei-swr-credential-id` | Username/Password | Huawei SWR image push |
| `dockerhub-credential-id` | Username/Password | Docker Hub image push |
| `dockerhub-oci-credentials` | Username/Password | `helm dep update` (OCI) |
| `kubeconfig-dev` | Secret File | Deploy to dev cluster |
| `kubeconfig-uat` | Secret File | Deploy to uat cluster |
| `kubeconfig-stag` | Secret File | Deploy to staging cluster |
| `kubeconfig-prod` | Secret File | Deploy to production cluster |
| `jenkins-api-user` | Username/Password | Create Jenkins jobs via API |

---

## 🔄 Deploy Pipeline Stages

```
Checkout          →   Clone service repo to service-src/
Build & Push      →   docker build + push with auto-generated IMAGE_TAG
Prepare Helm      →   Clone this repo, helm dep update, patch Chart.yaml version
Deploy            →   helm lint → dry-run → helm upgrade --atomic
Post-Deploy Check →   kubectl rollout status + pod list + recent events
```

If deploy fails, an automatic rollback is triggered via `helm rollback`.

---

## ✏️ Updating a Service Manually

Since Helm charts and `values.yaml` live directly in this repo, you can update them without re-running the generator:

```bash
# Clone the repo
git clone git@github.com:alfirmS/monorepo-devsecops.git
cd monorepo-devsecops

# Edit values for a service
vim services/my-service/helm-my-service/values.yaml

# Commit and push
git add .
git commit -m "fix: adjust resource limits for my-service"
git push origin main
```

The next deploy job run will automatically pick up the changes.

---

## 📐 Helm Chart Features

Each generated Helm chart supports:

- 🔁 **HPA** — HorizontalPodAutoscaler with CPU & memory targets
- 🛡️ **PDB** — PodDisruptionBudget for high availability
- 🔒 **ExternalSecret** — Vault KV v2 integration via External Secrets Operator
- 🌐 **Ingress** — Nginx Ingress with optional traffic splitting
- 📦 **Common Library** — Shared templates via OCI dependency (`oci://registry-1.docker.io/sfirman87/common-library`)

---

<div align="center">

Made with ☕ and way too many `helm upgrade` commands.

</div>
