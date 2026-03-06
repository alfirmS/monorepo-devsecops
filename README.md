<div align="center">

# 🎯 helm-starter

**Base template Helm chart untuk semua service di monorepo-devsecops.**

Branch ini adalah **sumber tunggal** untuk scaffolding Helm chart baru. Setiap kali service baru di-onboard via Jenkins Service Generator, chart ini di-copy, di-rename, dan di-patch otomatis ke `services/<service>/<env>/helm-<service>/`.

</div>

---

## 🗂️ Struktur

```
starter-helm/
├── Chart.yaml          # Metadata chart — name & description pakai {{ SERVICE }} placeholder
├── values.yaml         # Default values — semua field pakai {{ PLACEHOLDER }}
└── templates/
    ├── deployment.yaml       # Kubernetes Deployment dengan lifecycle hooks
    ├── service.yaml          # Kubernetes Service (ClusterIP / LoadBalancer / NodePort)
    ├── ingress.yaml          # Ingress atau HTTPRoute (Gateway API)
    ├── hpa.yaml              # HorizontalPodAutoscaler
    ├── pdb.yaml              # PodDisruptionBudget
    ├── externalsecrets.yaml  # ExternalSecret (Vault via ESO)
    └── serviceaccount.yaml   # ServiceAccount
```

---

## 📄 Chart.yaml

```yaml
apiVersion: v2
name: {{ SERVICE }}
description: |
  Helm chart untuk {{ SERVICE }} service.
type: application
version: 0.1.0
appVersion: "0.1.0"

dependencies:
  - name: common-library
    version: "1.0.0"
    repository: "oci://registry-1.docker.io/sfirman87"
    # Untuk dev lokal:
    # repository: "file://../commonLibrary"
```

| Field | Keterangan |
|---|---|
| `name` | Di-replace otomatis dengan nama service saat generator jalan. `{{ SERVICE }}` → `vampi` |
| `description` | Di-replace otomatis dengan nama service. |
| `version` | Di-patch pipeline saat deploy menjadi `ddmmyy.BUILD_NUMBER` |
| `appVersion` | Di-patch pipeline saat deploy menjadi image tag `commitId.ddmmyy.BUILD_NUMBER` |
| `dependencies` | Common library chart dari OCI registry — berisi helper templates yang di-share antar service |

### Common Library

Chart ini bergantung pada `common-library` yang di-publish ke Docker Hub OCI registry:

```
oci://registry-1.docker.io/sfirman87/common-library:1.0.0
```

Dependency ini di-pull otomatis via `helm dep update` di stage **Prepare Helm** pipeline, menggunakan credential `HELM_OCI_CRED_ID`.

Untuk development lokal, uncomment baris berikut di `Chart.yaml`:
```yaml
# repository: "file://../commonLibrary"
```

---

## ⚙️ Cara Kerja Generator

Saat **Jenkins Service Generator** dijalankan, berikut yang terjadi pada branch ini:

```
1. Clone branch helm-starter → helm-starter-tmp/

2. Copy isi starter-helm/ ke services/<service>/<env>/helm-<service>/
   cp -r helm-starter-tmp/starter-helm/. <helmDestDir>/

3. Replace {{ SERVICE }} placeholder di Chart.yaml
   "name: {{ SERVICE }}" → "name: vampi"

4. Patch values.yaml dengan parameter dari generator
   {{ CONTAINER_PORT }}  → 5000
   {{ HPA_ENABLED }}     → true
   {{ RESOURCE_PRESET }} → medium
   ... dst

5. Commit hasil ke branch main monorepo
```

---

## 📦 Templates

### `deployment.yaml`

Kubernetes Deployment dengan:
- **RollingUpdate strategy** — zero-downtime deploy (`maxUnavailable: 0`)
- **preStop lifecycle hook** — `sleep 5` untuk graceful shutdown sebelum pod di-remove dari load balancer
- **terminationGracePeriodSeconds** — buffer waktu untuk koneksi existing selesai
- **imagePullSecrets** — referensi ke `regcred` untuk pull dari private registry
- **Resource limits & requests** — dikontrol via `values.yaml`
- **Liveness & Readiness probe** — opsional, dikontrol via `healthCheck.enabled`

---

### `service.yaml`

Kubernetes Service yang mengekspos pod ke dalam cluster atau ke luar:

| `service.type` | Keterangan |
|---|---|
| `ClusterIP` | Hanya accessible di dalam cluster (default untuk backend) |
| `LoadBalancer` | Expose ke luar via cloud load balancer |
| `NodePort` | Expose via port di setiap node |

---

### `ingress.yaml`

Mendukung dua mode routing, dikontrol via `values.yaml`:

**Mode 1 — Ingress (Nginx, Traefik, dll):**
```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
```

**Mode 2 — HTTPRoute (Kubernetes Gateway API):**
```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway
      sectionName: http
  hostnames:
    - myapp.example.com
```

> Hanya satu yang bisa aktif dalam satu waktu.

---

### `hpa.yaml`

HorizontalPodAutoscaler dengan:
- Scale berdasarkan CPU dan Memory utilization
- **Scale-up behavior** — agresif (tambah 2 pod per menit)
- **Scale-down behavior** — konservatif (kurangi 1 pod per 2 menit, stabilization 5 menit)

```yaml
hpa:
  enabled: true       # dari parameter HPA_ENABLED
  minReplicas: 2      # dari parameter HPA_MIN_REPLICAS
  maxReplicas: 5      # dari parameter HPA_MAX_REPLICAS
  metrics:
    cpu:    70%       # dari parameter HPA_CPU_TARGET
    memory: 80%       # dari parameter HPA_MEM_TARGET
```

---

### `pdb.yaml`

PodDisruptionBudget — memastikan service tetap available saat cluster maintenance (node drain, upgrade):

```yaml
pdb:
  enabled: true       # dari parameter PDB_ENABLED
  minAvailable: 1     # dari parameter PDB_MIN_AVAILABLE
```

Tanpa PDB, semua pod bisa di-evict sekaligus saat node drain → **service down total**.

---

### `externalsecrets.yaml`

Integrasi dengan **External Secrets Operator (ESO)** untuk sync secret dari **Vault KV v2** ke Kubernetes Secret:

```yaml
externalSecret:
  enabled: false      # dari parameter EXTERNAL_SECRET_ENABLED
```

Jika diaktifkan, pastikan:
1. ESO sudah terinstall di cluster
2. `ClusterSecretStore` atau `SecretStore` sudah dikonfigurasi dengan Vault credentials
3. Path secret di Vault sudah sesuai dengan yang dikonfigurasi di template

---

### `serviceaccount.yaml`

Kubernetes ServiceAccount per-service:

```yaml
serviceAccount:
  create: true
  automount: true
  annotations: {}   # Bisa dipakai untuk IRSA (EKS) atau Workload Identity (GKE)
```

---

## 🔧 Update Starter Chart

Perubahan pada branch ini akan mempengaruhi **semua service yang di-generate setelah perubahan**. Service yang sudah ada **tidak** terpengaruh karena chart sudah di-copy ke folder masing-masing.

```bash
# Clone branch helm-starter
git clone --branch helm-starter git@github.com:alfirmS/monorepo-devsecops.git
cd monorepo-devsecops

# Edit template
vim starter-helm/templates/deployment.yaml

# Commit dan push
git add .
git commit -m "feat: add topologySpreadConstraints to deployment template"
git push origin helm-starter
```

---

## 🚀 Development Lokal

Untuk test chart secara lokal sebelum di-generate:

```bash
# Install dependency lokal
helm dependency update starter-helm/

# Lint chart
helm lint starter-helm/ -f starter-helm/values.yaml

# Dry-run render template
helm template my-service starter-helm/ \
  -f starter-helm/values.yaml \
  --set image.repository=myregistry/myapp \
  --set image.tag=latest

# Debug output
helm template my-service starter-helm/ \
  -f starter-helm/values.yaml \
  --debug 2>&1 | less
```

> Untuk dependency lokal (tanpa OCI pull), gunakan `repository: "file://../commonLibrary"` di `Chart.yaml`.

---

# 📄 values.yaml — Helm Chart Configuration Reference

File ini di-generate otomatis oleh **Jenkins Service Generator** dan berisi semua konfigurasi deployment untuk satu service di satu environment. Nilai-nilainya bisa diedit manual langsung di repo untuk fine-tuning tanpa perlu re-generate.

---

## 🔧 Top-Level

```yaml
replicaCount: 1
```

| Field | Tipe | Keterangan |
|---|---|---|
| `replicaCount` | `int` | Jumlah replica pod saat pertama deploy. Jika HPA aktif, nilai ini hanya berlaku saat initial deploy — selanjutnya HPA yang mengontrol. |

---

## 🌍 Global

```yaml
global:
  env: "uat"
```

| Field | Tipe | Nilai | Keterangan |
|---|---|---|---|
| `global.env` | `string` | `development` / `uat` / `staging` / `production` | Digunakan sebagai label environment di resources Kubernetes. Di-set otomatis oleh generator. |

---

## 🐳 Image

```yaml
image:
  repository: ""
  pullPolicy: Always
  tag: ""
```

| Field | Tipe | Keterangan |
|---|---|---|
| `image.repository` | `string` | URL registry + nama image. Dikosongkan di values, di-set via `--set image.repository=...` saat `helm upgrade`. |
| `image.pullPolicy` | `string` | `Always` / `IfNotPresent` / `Never`. Default `Always` untuk memastikan image terbaru selalu di-pull. |
| `image.tag` | `string` | Image tag. Dikosongkan di values, di-set via `--set image.tag=...` saat deploy menggunakan `IMAGE_TAG` dari pipeline. |

> **Catatan:** `repository` dan `tag` sengaja dikosongkan di file ini — nilai aktualnya di-inject oleh pipeline via `--set` flags saat `helm upgrade`.

---

## 🔑 Image Pull Secrets

```yaml
imagePullSecrets:
  - name: regcred
```

| Field | Keterangan |
|---|---|
| `imagePullSecrets` | Kubernetes secret bertipe `docker-registry` yang digunakan untuk pull image dari private registry. Secret `regcred` harus sudah dibuat di namespace yang sama sebelum deploy. |

Cara membuat `regcred`:
```bash
kubectl create secret docker-registry regcred \
  --docker-server=<REGISTRY_HOST> \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD> \
  --namespace=<ENV>
```

---

## 🏷️ Name Override

```yaml
nameOverride: ""
fullnameOverride: ""
```

| Field | Keterangan |
|---|---|
| `nameOverride` | Override nama chart (mengubah suffix resource name). Kosongkan untuk menggunakan nama chart default. |
| `fullnameOverride` | Override nama lengkap semua resource. Kosongkan untuk menggunakan format `release-chart`. |

---

## 👤 Service Account

```yaml
serviceAccount:
  create: true
  automount: true
  annotations: {}
```

| Field | Tipe | Keterangan |
|---|---|---|
| `serviceAccount.create` | `bool` | Buat ServiceAccount baru. Set `false` untuk menggunakan SA yang sudah ada. |
| `serviceAccount.automount` | `bool` | Auto-mount service account token ke pod. |
| `serviceAccount.annotations` | `map` | Annotation tambahan, misalnya untuk IRSA (IAM Role for Service Account) di EKS. |

---

## 🌐 Service

```yaml
service:
  type: LoadBalancer
  containerPort: 5000
  targetPort: 5000
```

| Field | Tipe | Nilai | Keterangan |
|---|---|---|---|
| `service.type` | `string` | `ClusterIP` / `LoadBalancer` / `NodePort` | Tipe Kubernetes Service. Di-set dari parameter `K8S_SERVICE_TYPE` saat generate. |
| `service.containerPort` | `int` | — | Port yang di-expose oleh container. Di-set dari parameter `CONTAINER_PORT` saat generate. |
| `service.targetPort` | `int` | — | Port yang di-forward ke container (biasanya sama dengan `containerPort`). |

---

## 🔀 Ingress

```yaml
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
```

| Field | Tipe | Keterangan |
|---|---|---|
| `ingress.enabled` | `bool` | Enable/disable Ingress resource. |
| `ingress.className` | `string` | IngressClass, e.g. `nginx`. |
| `ingress.annotations` | `map` | Annotations untuk ingress controller, e.g. `nginx.ingress.kubernetes.io/rewrite-target`. |
| `ingress.hosts` | `list` | Daftar hostname dan path mapping. |
| `ingress.tls` | `list` | TLS configuration, referensi ke secret certificate. |

Contoh konfigurasi dengan TLS:
```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com
```

---

## 🔀 HTTP Route (Gateway API)

```yaml
httpRoute:
  enabled: false
  annotations: {}
  parentRefs:
    - name: gateway
      sectionName: http
  hostnames:
    - chart-example.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

Alternatif Ingress menggunakan **Kubernetes Gateway API**. Aktifkan jika cluster menggunakan Gateway API controller (e.g. Istio, Envoy Gateway).

| Field | Keterangan |
|---|---|
| `httpRoute.enabled` | Enable/disable HTTPRoute resource. Tidak bisa aktif bersamaan dengan `ingress.enabled`. |
| `httpRoute.parentRefs` | Gateway yang menjadi parent route ini. |
| `httpRoute.hostnames` | Daftar hostname yang di-handle. |
| `httpRoute.rules` | Routing rules — path matching, header matching, dll. |

---

## 💾 Resources

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"
```

| Field | Keterangan |
|---|---|
| `resources.limits.cpu` | Maksimum CPU yang boleh digunakan pod. Jika terlampaui, pod di-throttle. |
| `resources.limits.memory` | Maksimum memory. Jika terlampaui, pod di-OOMKill. |
| `resources.requests.cpu` | CPU yang di-reserve untuk scheduling. |
| `resources.requests.memory` | Memory yang di-reserve untuk scheduling. |

Di-set otomatis dari parameter `RESOURCE_PRESET` saat generate:

| Preset | CPU Limit | CPU Request | Memory Limit | Memory Request |
|---|---|---|---|---|
| `small` | `500m` | `250m` | `512Mi` | `256Mi` |
| `medium` | `1` | `500m` | `1Gi` | `512Mi` |
| `large` | `2` | `1` | `2Gi` | `1Gi` |
| `xlarge` | `4` | `2` | `4Gi` | `2Gi` |

---

## ❤️ Health Check

```yaml
healthCheck:
  enabled: false
livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http
```

| Field | Keterangan |
|---|---|
| `healthCheck.enabled` | Toggle master untuk liveness & readiness probe. Set `true` dan sesuaikan path jika service punya health endpoint. |
| `livenessProbe` | Probe untuk mendeteksi apakah pod perlu di-restart. |
| `readinessProbe` | Probe untuk menentukan apakah pod siap menerima traffic. |

Contoh untuk service dengan endpoint `/health`:
```yaml
healthCheck:
  enabled: true
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 15
  periodSeconds: 20
readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## 🚀 Deployment

```yaml
deployment:
  enabled: true
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 5"]
  terminationGracePeriodSeconds: 30
```

| Field | Keterangan |
|---|---|
| `deployment.enabled` | Toggle deployment resource. |
| `deployment.strategy.type` | `RollingUpdate` (zero-downtime) atau `Recreate` (downtime, cocok untuk dev). |
| `deployment.strategy.rollingUpdate.maxSurge` | Jumlah pod extra yang boleh dibuat saat rolling update. |
| `deployment.strategy.rollingUpdate.maxUnavailable` | Jumlah pod yang boleh tidak available saat update. `0` = zero-downtime. |
| `deployment.minReadySeconds` | Waktu tunggu (detik) setelah pod Ready sebelum dianggap available. Buffer untuk stabilitas. |
| `deployment.lifecycle.preStop` | Hook yang dijalankan sebelum container dihentikan. `sleep 5` memberi waktu load balancer untuk remove pod dari pool sebelum koneksi diputus. |
| `deployment.terminationGracePeriodSeconds` | Total waktu Kubernetes menunggu pod berhenti dengan baik sebelum di-force kill. |

---

## 📈 HPA — HorizontalPodAutoscaler

```yaml
hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

| Field | Keterangan |
|---|---|
| `hpa.enabled` | Enable HorizontalPodAutoscaler. Di-set dari parameter `HPA_ENABLED` saat generate. |
| `hpa.minReplicas` | Jumlah minimum replica saat load rendah. |
| `hpa.maxReplicas` | Jumlah maximum replica saat load tinggi. |
| `hpa.behavior.scaleUp.stabilizationWindowSeconds` | Waktu tunggu sebelum scale-up dieksekusi (hindari flapping). |
| `hpa.behavior.scaleDown.stabilizationWindowSeconds` | Waktu tunggu sebelum scale-down (lebih konservatif untuk hindari thrashing). |
| `hpa.metrics[].averageUtilization` | Threshold utilization (%). Scale-up terjadi jika rata-rata melebihi nilai ini. |

> **Tips:** `scaleDown.stabilizationWindowSeconds: 300` (5 menit) adalah nilai aman untuk production — mencegah scale-down terlalu agresif saat spike traffic pendek.

---

## 🛡️ PDB — PodDisruptionBudget

```yaml
pdb:
  enabled: true
  minAvailable: 1
```

| Field | Keterangan |
|---|---|
| `pdb.enabled` | Enable PodDisruptionBudget. Di-set dari parameter `PDB_ENABLED` saat generate. |
| `pdb.minAvailable` | Minimum jumlah pod yang harus tetap available saat ada disruption (node drain, cluster upgrade, dll). Bisa angka (`1`) atau persentase (`"50%"`). |

> **PDB penting untuk production** — memastikan setidaknya 1 pod tetap berjalan saat maintenance cluster, sehingga service tidak down total.

---

## 🔒 External Secret

```yaml
externalSecret:
  enabled: false
```

| Field | Keterangan |
|---|---|
| `externalSecret.enabled` | Enable External Secrets Operator (ESO) untuk sync secret dari Vault KV v2. Di-set dari parameter `EXTERNAL_SECRET_ENABLED` saat generate. |

Jika diaktifkan, chart akan membuat resource `ExternalSecret` yang otomatis men-sync secret dari Vault ke Kubernetes secret. Pastikan **External Secrets Operator** sudah terinstall di cluster dan `SecretStore` / `ClusterSecretStore` sudah dikonfigurasi.

---

## 📦 Volumes & Extra Config

```yaml
volumes: []
volumeMounts: []
nodeSelector: {}
tolerations: []
affinity: {}
```

| Field | Keterangan |
|---|---|
| `volumes` | Tambahan volume (ConfigMap, Secret, emptyDir, PVC, dll). |
| `volumeMounts` | Mount point tambahan di container. |
| `nodeSelector` | Schedule pod ke node dengan label tertentu. |
| `tolerations` | Tolerasi untuk node yang punya taint. |
| `affinity` | Pod affinity/anti-affinity rules — berguna untuk spread pod ke multiple node. |

Contoh anti-affinity (spread pod ke node berbeda):
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - my-service
          topologyKey: kubernetes.io/hostname
```

---

## 📝 Cara Edit Manual

```bash
# Edit values untuk service tertentu
vim services/vampi/uat/helm-vampi/values.yaml

# Commit dan push
git add services/vampi/uat/helm-vampi/values.yaml
git commit -m "fix: update HPA maxReplicas vampi uat"
git push origin main
```

Pipeline deploy berikutnya akan otomatis menggunakan values terbaru tanpa perlu re-generate.

<div align="center">

Branch ini adalah fondasi — ubah dengan hati-hati, test dulu sebelum push. 🧱

</div>
