# Jenkinsfile.generator

Pipeline Jenkins untuk generate struktur service baru di monorepo secara otomatis.

Satu kali run menghasilkan:
```
services/<service>/<env>/
├── Jenkinsfile
├── Dockerfile
└── helm-<service>/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

---

## Parameter

### Service

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `SERVICE` | string | — | Nama service, lowercase alphanumeric & hyphens. e.g. `payment-api` |
| `ENV` | choice | `dev` | Target environment: `dev`, `uat`, `stag`, `prod` |
| `CONTAINER_PORT` | string | `8080` | Port yang di-expose container |
| `K8S_SERVICE_TYPE` | choice | `ClusterIP` | Kubernetes Service type |

### Resource

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `RESOURCE_PRESET` | choice | `small` | `small` / `medium` / `large` / `xlarge` / `custom` |
| `CUSTOM_CPU_LIMIT` | string | `500m` | Aktif hanya jika preset `custom` |
| `CUSTOM_CPU_REQUEST` | string | `250m` | Aktif hanya jika preset `custom` |
| `CUSTOM_MEM_LIMIT` | string | `512Mi` | Aktif hanya jika preset `custom` |
| `CUSTOM_MEM_REQUEST` | string | `256Mi` | Aktif hanya jika preset `custom` |

Resource preset:

| Preset | CPU limit | CPU request | Memory limit | Memory request |
|---|---|---|---|---|
| `small` | 500m | 250m | 512Mi | 256Mi |
| `medium` | 1 | 500m | 1Gi | 512Mi |
| `large` | 2 | 1 | 2Gi | 1Gi |
| `xlarge` | 4 | 2 | 4Gi | 2Gi |

### HPA

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `HPA_ENABLED` | boolean | `false` | Enable HorizontalPodAutoscaler |
| `HPA_MIN_REPLICAS` | string | `1` | — |
| `HPA_MAX_REPLICAS` | string | `10` | — |
| `HPA_CPU_TARGET` | string | `70` | Target CPU utilization (%) |
| `HPA_MEM_TARGET` | string | `80` | Target Memory utilization (%) |

### PDB

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `PDB_ENABLED` | boolean | `false` | Enable PodDisruptionBudget |
| `PDB_MIN_AVAILABLE` | string | `1` | Angka atau persen, e.g. `1` atau `50%` |

### Registry

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `REGISTRY_TYPE` | choice | `Harbor` | `Harbor` / `Huawei SWR` / `Docker Hub` |
| `REGISTRY_HOST` | string | — | URL registry, e.g. `harbor.example.com` |
| `REGISTRY_SECRET` | string | `regcred` | Nama Kubernetes secret `docker-registry` di namespace jenkins |
| `HELM_OCI_REGISTRY_TYPE` | choice | `Docker Hub` | Registry untuk helm chart dependency |
| `EXTERNAL_SECRET_ENABLED` | boolean | `false` | Enable ExternalSecret (Vault integration) |

### Dockerfile

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `GENERATE_DOCKERFILE` | boolean | `true` | Generate `Dockerfile` ke folder service |
| `DOCKERFILE_RUNTIME_TYPE` | choice | `node-npm` | Lihat tabel runtime di bawah |
| `VAULT_ADDR` | string | — | **Khusus `vuejs`** — URL Vault, e.g. `https://vault.example.com` |
| `VAULT_SECRET_PATH` | string | — | **Khusus `vuejs`** — path secret, e.g. `secret/data/myapp/uat` |
| `VAULT_CRED_ID` | string | `vault-token` | **Khusus `vuejs`** — Jenkins credential ID (Secret Text) |

Runtime type yang tersedia:

| Value | Builder | Runtime | Catatan |
|---|---|---|---|
| `node-npm` | `node:20-alpine` | `node:20-alpine` | `npm ci` + `npm run build` |
| `node-yarn` | `node:20-alpine` | `node:20-alpine` | `yarn install` + `yarn build` |
| `vuejs` | `node:20-alpine` | `node:20-alpine` | Vault `.env` inject + `npm run build` + `npm run start` |
| `python-pip` | `python:3.12-slim` | `python:3.12-slim` | `pip install` |
| `java-jar` | `maven:3.9-temurin` | `distroless/java21` | `mvn package` |
| `golang` | `golang:1.22-alpine` | `distroless/static` | `go build` |
| `static-nginx` | `node:20-alpine` | `nginx:1.27-alpine` | `npm run build` → static |
| `custom-cmd` | `alpine:3.19` | `alpine:3.19` | Manual — edit template langsung |

> Template Dockerfile ada di `templates/dockerfile/Dockerfile.<runtime>.template`.
> Nama file template harus sinkron dengan nilai `DOCKERFILE_RUNTIME_TYPE`.

### Git & Jenkins

| Parameter | Tipe | Default | Keterangan |
|---|---|---|---|
| `GIT_CRED_ID` | choice | `git-credential-id` | Jenkins credential ID untuk akses Git (SSH) |
| `JENKINS_FOLDER` | string | — | Folder Jenkins tempat job dibuat. Kosong = root. e.g. `backend/java` |

---

## Stages

```
1. Validate Parameters   → cek SERVICE tidak kosong, format valid
2. Clone DevOps Repo     → clone monorepo-devsecops branch main
3. Clone Starter Helm    → clone branch helm-starter untuk template chart
4. Prepare Service Dir   → buat services/<service>/<env>/, copy helm starter
5. Patch Values & Generate → patch values.yaml, generate Jenkinsfile + Dockerfile
6. Push to DevOps Repo   → git commit + push hasil generate
7. Create Jenkins Job    → buat atau update job via Jenkins API
```

---

## Kubernetes Pod

Pipeline berjalan di Kubernetes dengan 2 container:

| Container | Image | Dipakai untuk |
|---|---|---|
| `jnlp` | auto-inject plugin | `readFile`, `writeFile`, logic Groovy |
| `tools` | `alpine/git` | git clone, git push, curl Jenkins API |

---

## Credential yang Dibutuhkan

| Credential ID | Tipe | Dipakai untuk |
|---|---|---|
| `git-credential-id` / `github-ssh-key` / `bitbucket-scm-deployer` | SSH Private Key | Clone & push monorepo |
| `harbor-credential-id` / `huawei-credential-id` / `dockerhub-credential-id` | Username/Password | Push image ke registry |
| `kubeconfig-dev/uat/stag/prod` | File | Deploy ke cluster |
| `jenkins-api-user` | Username/Password | Buat job via Jenkins API |
| `vault-token` | Secret Text | Fetch secret Vault (khusus vuejs) |

---

## Output

Setelah pipeline selesai, hasil generate ada di:

```
services/<service>/<env>/
├── Jenkinsfile                        ← pipeline deploy service
├── Dockerfile                         ← build image (dari template runtime)
└── helm-<service>/
    ├── Chart.yaml                     ← nama & versi chart
    ├── values.yaml                    ← resource, HPA, PDB, port, dll
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── hpa.yaml
        ├── pdb.yaml
        ├── externalsecrets.yaml
        └── serviceaccount.yaml
```

Jenkins job deploy otomatis dibuat di: `<JENKINS_FOLDER>/<SERVICE>`
