<div align="center">

# 🚀 monorepo-devsecops

**One repo to rule all deployments.**

Centralized GitOps monorepo untuk Helm charts, Dockerfile, dan Jenkinsfile — di-generate dan di-maintain otomatis oleh Jenkins Service Generator, di-deploy via Kubernetes-native agent menggunakan Kaniko + Helm.

[![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![Helm](https://img.shields.io/badge/Packaging-Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Kaniko](https://img.shields.io/badge/Build-Kaniko-F5A623?style=for-the-badge&logo=docker&logoColor=white)](https://github.com/GoogleContainerTools/kaniko)
[![Vault](https://img.shields.io/badge/Secrets-Vault-FFEC6E?style=for-the-badge&logo=vault&logoColor=black)](https://www.vaultproject.io/)

</div>

---

## 📖 Overview

Repo ini adalah **single source of truth** untuk seluruh konfigurasi deployment. Setiap service punya folder sendiri yang berisi Jenkinsfile, Dockerfile, dan Helm chart — semuanya di-generate otomatis oleh pipeline generator dan di-commit langsung ke sini.

Pipeline deploy berjalan **100% di dalam Kubernetes** menggunakan Pod agent dengan dua container: **Kaniko** (build image tanpa Docker daemon) dan **Helm** (deploy ke cluster).

---


---

## ✅ Prerequisites

Sebelum menggunakan monorepo ini, pastikan semua komponen berikut sudah tersedia dan terkonfigurasi.

### 1. Kubernetes Cluster

Jenkins agent berjalan sebagai Pod di dalam cluster. Pastikan:

- Kubernetes cluster sudah berjalan (v1.24+)
- Jenkins sudah terinstall dan terhubung ke cluster via **Kubernetes plugin**
- Jenkins service account punya permission untuk create Pod di namespace `jenkins`

```bash
# Verifikasi koneksi Jenkins ke cluster
kubectl get pods -n jenkins
```

### 2. Jenkins Plugins

Install plugin berikut di Jenkins (**Manage Jenkins → Plugins**):

| Plugin | Keterangan |
|---|---|
| **Kubernetes** | Menjalankan build agent sebagai Pod di cluster |
| **Git** | Clone repository |
| **GitHub** | GitHub webhook & `githubPush()` trigger |
| **Credentials Binding** | Inject credentials ke pipeline |
| **Pipeline** | Declarative pipeline support |

### 3. Kubernetes Secret untuk Registry (regcred)

Kaniko membaca credential registry dari `/kaniko/.docker/config.json` yang di-mount dari Kubernetes secret. Buat secret ini di namespace `jenkins` **sebelum** menjalankan pipeline apapun:

```bash
# Untuk Harbor / private registry
kubectl create secret docker-registry regcred \
  --docker-server=<REGISTRY_HOST> \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD> \
  --namespace=jenkins

# Untuk Docker Hub
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<DOCKERHUB_USER> \
  --docker-password=<DOCKERHUB_TOKEN> \
  --namespace=jenkins

# Verifikasi isi secret
kubectl get secret regcred -n jenkins \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

> **Catatan:** Nama secret default adalah `regcred`. Jika menggunakan nama lain, sesuaikan parameter `REGISTRY_SECRET` saat menjalankan generator.

### 4. Jenkins Credentials

Tambahkan credentials berikut di **Manage Jenkins → Credentials**:

| Credential ID | Type | Cara Membuat |
|---|---|---|
| `git-credential-id` | SSH Username with private key | Generate SSH key, tambahkan public key ke GitHub Deploy Keys |
| `harbor-credential-id` | Username with password | Username & password Harbor registry |
| `huawei-credential-id` | Username with password | AK/SK Huawei SWR |
| `dockerhub-credential-id` | Username with password | Username & access token Docker Hub |
| `kubeconfig-dev` | Secret file | File kubeconfig cluster dev |
| `kubeconfig-uat` | Secret file | File kubeconfig cluster uat |
| `kubeconfig-stag` | Secret file | File kubeconfig cluster staging |
| `kubeconfig-prod` | Secret file | File kubeconfig cluster production |
| `vault-token` | Secret text | Token Vault (hanya jika pakai VueJS + Vault) |
| `jenkins-api-user` | Username with password | Username & API token Jenkins (User → Configure → API Token) |

```bash
# Generate SSH key untuk git-credential-id
ssh-keygen -t ed25519 -C "jenkins@ci" -f ~/.ssh/jenkins_deploy_key

# Tambahkan public key ke GitHub repo → Settings → Deploy Keys
cat ~/.ssh/jenkins_deploy_key.pub

# Private key dimasukkan ke Jenkins credential
cat ~/.ssh/jenkins_deploy_key
```

### 5. GitHub Webhook (untuk auto-trigger)

Agar `githubPush()` trigger berfungsi di environment `dev`, `uat`, dan `stag`:

1. Buka repository service di GitHub → **Settings → Webhooks → Add webhook**
2. Payload URL: `http://<JENKINS_URL>/github-webhook/`
3. Content type: `application/json`
4. Events: **Just the push event** ✅
5. Active: ✅

### 6. Helm Starter Chart

Branch `helm-starter` harus berisi folder `starter-helm/` yang merupakan base Helm chart untuk semua service baru.

```bash
# Verifikasi branch helm-starter
git clone --branch helm-starter git@github.com:alfirmS/monorepo-devsecops.git helm-starter-check
ls helm-starter-check/starter-helm/
# Harus ada: Chart.yaml  values.yaml  templates/
```

### 7. Jenkinsfile Generator Job

Buat pipeline job di Jenkins yang mengarah ke `templates/generator/Jenkinsfile.generator`:

1. Jenkins → **New Item** → **Pipeline**
2. Name: `service-generator`
3. Definition: **Pipeline script from SCM**
4. SCM: Git → `git@github.com:alfirmS/monorepo-devsecops.git`
5. Credentials: `git-credential-id`
6. Branch: `main`
7. Script Path: `templates/generator/Jenkinsfile.generator`

---

## 🗂️ Struktur Repo

```
monorepo-devsecops/
├── templates/
│   ├── Jenkinsfile.template          # Template pipeline deploy (semua service)
│   ├── Dockerfile.template           # Generic Dockerfile template
│   ├── config.xml                    # Jenkins job XML template
│   ├── job-config.xml
│   ├── dockerfile/
│   │   ├── Dockerfile.golang.template
│   │   ├── Dockerfile.nodejs.template
│   │   ├── Dockerfile.python.template
│   │   └── Dockerfile.vuejs.template
│   └── generator/
│       ├── Jenkinsfile.generator     # Pipeline generator (taruh di Jenkins)
│       └── README.generator.md
│
└── services/
    └── <service>/
        └── <env>/                    # dev | uat | stag | prod
            ├── Jenkinsfile           # Auto-generated deploy pipeline
            ├── Dockerfile            # Auto-generated (opsional)
            └── helm-<service>/
                ├── Chart.yaml
                ├── values.yaml       # Patched per environment
                └── templates/
                    ├── deployment.yaml
                    ├── service.yaml
                    ├── ingress.yaml
                    ├── hpa.yaml
                    ├── pdb.yaml
                    ├── externalsecrets.yaml
                    └── serviceaccount.yaml
```

### Services yang sudah terdaftar

| Service | Environment | Helm Chart |
|---|---|---|
| `ghost` | `uat` | `helm-ghost` |
| `juice-shop` | `uat` | `helm-juice-shop` |
| `vampi` | `uat` | `helm-vampi` |

---

## 🌿 Branch Strategy

| Branch | Isi | Keterangan |
|---|---|---|
| `main` | Templates, services, generated files | Branch utama — semua commit generator masuk ke sini |
| `helm-starter` | `starter-helm/` | Base Helm chart template untuk scaffolding service baru |

---

## ⚙️ Cara Kerja

```
┌──────────────────────────────────────────────────────────┐
│               Jenkins Service Generator                   │
│          (templates/generator/Jenkinsfile.generator)      │
│                                                           │
│  1. Validate parameter SERVICE, ENV, dll                  │
│  2. Clone monorepo (branch main) → devops-workspace/      │
│  3. Clone helm-starter → helm-starter-tmp/                │
│  4. Salin starter ke services/<svc>/<env>/helm-<svc>/     │
│  5. Patch values.yaml sesuai parameter                    │
│  6. Generate Jenkinsfile dari Jenkinsfile.template        │
│  7. Generate Dockerfile dari template runtime-spesifik    │
│  8. Commit & push ke monorepo                             │
│  9. Create/update Jenkins deploy job via API              │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│             Jenkins Deploy Job (auto-created)             │
│          (services/<svc>/<env>/Jenkinsfile)               │
│                                                           │
│  Pod Agent: kaniko + helm (running in Kubernetes)         │
│                                                           │
│  Checkout  → clone service-src + devops-repo             │
│  Build     → kaniko build & push image (no Docker daemon) │
│  Prepare   → helm dep update + patch Chart.yaml version  │
│  Deploy    → helm lint → dry-run → helm upgrade --atomic  │
│  Check     → rollout status + stability check (restart)   │
│  Rollback  → helm rollback otomatis jika gagal            │
└──────────────────────────────────────────────────────────┘
```

---

## 🏗️ Onboarding Service Baru

Jalankan **Jenkinsfile.generator** di Jenkins dengan parameter berikut:

### Identity & Networking

| Parameter | Deskripsi | Contoh |
|---|---|---|
| `SERVICE` | Nama service (lowercase, boleh hyphens) | `payment-api` |
| `ENV` | Target environment | `dev` / `uat` / `stag` / `prod` |
| `CONTAINER_PORT` | Port yang di-expose container | `8080` |
| `K8S_SERVICE_TYPE` | Kubernetes Service type | `ClusterIP` / `LoadBalancer` |

### Resources

| Parameter | Deskripsi |
|---|---|
| `RESOURCE_PRESET` | Pilih preset atau `custom` untuk manual |
| `CUSTOM_CPU_LIMIT` | CPU limit (jika preset = custom) |
| `CUSTOM_CPU_REQUEST` | CPU request (jika preset = custom) |
| `CUSTOM_MEM_LIMIT` | Memory limit (jika preset = custom) |
| `CUSTOM_MEM_REQUEST` | Memory request (jika preset = custom) |

### Fitur Opsional

| Parameter | Default | Deskripsi |
|---|---|---|
| `HPA_ENABLED` | `false` | Enable HorizontalPodAutoscaler |
| `HPA_MIN_REPLICAS` | `1` | HPA minimum replicas |
| `HPA_MAX_REPLICAS` | `10` | HPA maximum replicas |
| `HPA_CPU_TARGET` | `70` | Target CPU utilization (%) |
| `HPA_MEM_TARGET` | `80` | Target Memory utilization (%) |
| `PDB_ENABLED` | `false` | Enable PodDisruptionBudget |
| `PDB_MIN_AVAILABLE` | `1` | Minimum pod available (angka atau %) |
| `EXTERNAL_SECRET_ENABLED` | `false` | Enable Vault secret sync via ESO |

### Build & Registry

| Parameter | Deskripsi | Contoh |
|---|---|---|
| `REGISTRY_TYPE` | Container registry | `Harbor` / `Huawei SWR` / `Docker Hub` |
| `REGISTRY_HOST` | Registry URL | `registry.example.com` |
| `REGISTRY_SECRET` | K8s docker-registry secret di namespace jenkins | `regcred` |
| `HELM_OCI_REGISTRY_TYPE` | Registry untuk helm chart dependency (OCI) | `Docker Hub` / `Same as Image Registry` |

### Dockerfile

| Parameter | Deskripsi |
|---|---|
| `GENERATE_DOCKERFILE` | Generate Dockerfile ke `services/<svc>/<env>/Dockerfile` |
| `DOCKERFILE_RUNTIME_TYPE` | `node-npm` / `node-yarn` / `vuejs` / `python-pip` / `java-jar` / `golang` / `static-nginx` / `custom-cmd` |
| `VAULT_ADDR` | URL Vault (khusus vuejs dengan secret injection saat build) |
| `VAULT_SECRET_PATH` | Path secret di Vault, e.g. `secret/data/myapp/uat` |
| `VAULT_CRED_ID` | Jenkins credential ID (Secret Text) untuk Vault token |

### Git & Jenkins

| Parameter | Deskripsi |
|---|---|
| `GIT_CRED_ID` | Jenkins SSH credential untuk akses Git (`git-credential-id` / `github-ssh-key` / `bitbucket-scm-deployer`) |
| `JENKINS_FOLDER` | Folder Jenkins tempat deploy job dibuat (kosong = root), e.g. `backend/java` |

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

| ENV | Namespace | Default Branch | Auto Trigger | Kubeconfig Credential |
|---|---|---|---|---|
| `dev` | `dev` | `dev` | ✅ `githubPush()` | `kubeconfig-dev` |
| `uat` | `uat` | `uat` | ✅ `githubPush()` | `kubeconfig-uat` |
| `stag` | `stag` | `main` | ✅ `githubPush()` | `kubeconfig-stag` |
| `prod` | `prod` | manual input param | ❌ manual only | `kubeconfig-prod` |

> `prod` tidak punya auto-trigger dan memerlukan input branch/tag secara manual setiap kali deploy — sebagai safeguard production.

---

## 🏷️ Image & Chart Versioning

Image tag dibuat otomatis saat build:

```
{commitShortSha}.{ddmmyy}.{BUILD_NUMBER}

# Contoh:
a1b2c3d.060326.42
```

`Chart.yaml` di-patch sebelum deploy:

```yaml
version: "060326.42"            # dateStamp.buildNumber
appVersion: "a1b2c3d.060326.42" # image tag lengkap
```

---

## 🐳 Build dengan Kaniko

Image di-build menggunakan **Kaniko** — tidak memerlukan Docker daemon, aman dijalankan sebagai container biasa di dalam Kubernetes.

```
kaniko executor
  --context        dir://workspace/service-src
  --dockerfile     workspace/devops-repo/services/<svc>/<env>/Dockerfile
  --destination    <registry>/<service>:<tag>
  --cache          true
  --cache-ttl      24h
  --snapshot-mode  redo
```

Untuk service VueJS, Kaniko menerima `--build-arg VAULT_TOKEN` untuk secret injection saat build time.

---

## 🔐 Required Jenkins Credentials

| Credential ID | Type | Digunakan Untuk |
|---|---|---|
| `git-credential-id` | SSH Private Key | Clone & push ke GitHub |
| `harbor-credential-id` | Username/Password | Push image ke Harbor |
| `huawei-credential-id` | Username/Password | Push image ke Huawei SWR |
| `dockerhub-credential-id` | Username/Password | Push image ke Docker Hub |
| `kubeconfig-dev` | Secret File | Deploy ke cluster dev |
| `kubeconfig-uat` | Secret File | Deploy ke cluster uat |
| `kubeconfig-stag` | Secret File | Deploy ke cluster staging |
| `kubeconfig-prod` | Secret File | Deploy ke cluster production |
| `vault-token` | Secret Text | Vault token untuk build-time secret injection |
| `jenkins-api-user` | Username/Password | Create/update Jenkins job via API |

---

## 🔄 Deploy Pipeline Stages

```
Checkout          →  Clone service repo (service-src/)
                     Clone monorepo (devops-repo/)
Build & Push      →  kaniko build image → push ke registry dengan IMAGE_TAG
Prepare Helm      →  helm dep update (OCI)
                     Patch Chart.yaml: version = dateStamp.build, appVersion = imageTag
Deploy            →  helm lint → dry-run → helm upgrade --install --atomic --timeout 5m
Post-Deploy Check →  rollout status → sleep 30s → cek restart count
                     ↳ jika restart > 0: print logs + describe + events → error → rollback
```

Jika deploy gagal atau restart terdeteksi, pipeline otomatis menjalankan `helm rollback` ke release sebelumnya.

---

## 📋 Dockerfile Templates

Generator mendukung 8 runtime template yang tersedia di `templates/dockerfile/`:

| Runtime | Template File | Keterangan |
|---|---|---|
| `node-npm` | `Dockerfile.nodejs.template` | Node.js dengan npm |
| `node-yarn` | `Dockerfile.nodejs.template` | Node.js dengan yarn |
| `vuejs` | `Dockerfile.vuejs.template` | Vue.js + optional Vault secret injection |
| `python-pip` | `Dockerfile.python.template` | Python dengan pip |
| `golang` | `Dockerfile.golang.template` | Go binary multi-stage build |
| `java-jar` | — | Java executable JAR |
| `static-nginx` | — | Static files via Nginx |
| `custom-cmd` | — | Custom entrypoint |

---

## ✏️ Update Manual

Karena Helm chart dan `values.yaml` langsung ada di repo, kamu bisa edit tanpa re-generate:

```bash
git clone git@github.com:alfirmS/monorepo-devsecops.git
cd monorepo-devsecops

# Contoh: edit resource limit juice-shop di uat
vim services/juice-shop/uat/helm-juice-shop/values.yaml

git add .
git commit -m "fix: adjust memory limit juice-shop uat"
git push origin main
```

Deploy job berikutnya akan otomatis menggunakan konfigurasi terbaru.

---

<div align="center">

Made with ☕, `helm upgrade --atomic`, dan banyak iterasi.

</div>
