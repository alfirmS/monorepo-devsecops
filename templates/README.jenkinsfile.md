# Jenkinsfile.template

Template pipeline deploy per service. Di-generate otomatis oleh `Jenkinsfile.generator` dan disimpan di `services/<service>/<env>/Jenkinsfile`.

Jangan edit file ini langsung untuk mengubah pipeline satu service — edit template ini jika ingin mengubah perilaku **semua** service sekaligus, lalu jalankan ulang generator.

---

## Placeholder

Semua placeholder diisi oleh `Jenkinsfile.generator` saat generate.

| Placeholder | Contoh hasil |
|---|---|
| `{{ SERVICE }}` | `ghost` |
| `{{ ENV }}` | `uat` |
| `{{ ENV_UPPER }}` | `UAT` |
| `{{ SERVICE_REPO }}` | `git@github.com:org/ghost.git` |
| `{{ CHECKOUT_BRANCH }}` | `uat` / `main` / `${params.GIT_BRANCH}` |
| `{{ REGISTRY_HOST }}` | `harbor.example.com` |
| `{{ REGISTRY_CRED_ID }}` | `harbor-credential-id` |
| `{{ REGISTRY_SECRET }}` | `regcred` |
| `{{ KUBECONFIG_CRED_ID }}` | `kubeconfig-uat` |
| `{{ HELM_OCI_CRED_ID }}` | `dockerhub-credential-id` |
| `{{ GIT_CRED_ID }}` | `github-ssh-key` |
| `{{ VAULT_CRED_ID }}` | `vault-token` ← kosong jika bukan runtime `vuejs` |
| `{{ PARAMETERS_BLOCK }}` | Block `parameters { }` — hanya diisi untuk env `prod` |
| `{{ TRIGGER_BLOCK }}` | Block `triggers { githubPush() }` — diisi untuk semua env kecuali `prod` |

---

## Kubernetes Pod

Pipeline berjalan di Kubernetes dengan 3 container:

| Container | Image | Dipakai untuk |
|---|---|---|
| `jnlp` | auto-inject plugin | `checkout`, `script`, logic Groovy — `defaultContainer` |
| `kaniko` | `gcr.io/kaniko-project/executor:debug` | Build & push Docker image tanpa daemon |
| `helm` | `alpine/helm:latest` | `helm dep update`, `helm upgrade`, `kubectl` |

`kaniko` membaca kredensial registry dari projected volume `{{ REGISTRY_SECRET }}` yang di-mount ke `/kaniko/.docker/config.json`.

---

## Stages

### 1. Checkout
Checkout dua repo secara paralel:
- `service-src/` — source code service, branch sesuai `{{ CHECKOUT_BRANCH }}`
- `devops-repo/` — monorepo devops, branch `main` (berisi Dockerfile & Helm chart)

### 2. Build & Push Image
Image tag format: `<commitId>.<ddmmyy>.<buildNumber>` — contoh: `a1b2c3d.010125.42`

`commitId` dan `dateStamp` diambil di `jnlp` karena `kaniko:debug` tidak punya `git` dan `date`. Hasilnya di-set ke `env.IMAGE_TAG` lalu dipakai di container `kaniko`.

Untuk runtime `vuejs`: kaniko menerima `--build-arg VAULT_TOKEN` dari Jenkins credential `{{ VAULT_CRED_ID }}`. Untuk runtime lain, `VAULT_TOKEN` tidak di-pass — `ARG VAULT_TOKEN` tidak ada di Dockerfile-nya sehingkan kaniko mengabaikannya.

### 3. Prepare Helm
- Login ke Helm OCI registry
- `helm dep update` — download chart dependencies
- Logout dari registry
- Patch `Chart.yaml`: `version` dan `appVersion` di-set ke nilai runtime

`CHART_VERSION` format: `<ddmmyy>.<buildNumber>` — contoh: `010125.42`

### 4. Deploy
- `helm lint` — validasi chart
- `helm upgrade --install --dry-run` — simulasi deploy
- `helm upgrade --install --atomic --timeout 5m` — deploy sesungguhnya

Flag `--atomic`: jika deploy gagal, helm otomatis rollback ke revision sebelumnya. Flag `--history-max 5` membatasi histori revision yang disimpan.

### 5. Post-Deploy Check
Stability check setelah helm selesai karena `--atomic` hanya memastikan pod `Running`, belum tentu stabil.

Flow:
1. `kubectl rollout status` — tunggu rollout selesai
2. `sleep 30` — beri waktu pod untuk crash jika tidak stabil
3. Cek total `restartCount` semua pod
4. Jika restart > 0: cetak logs + describe pod, lalu `error` → trigger rollback di `post { failure }`
5. Jika restart = 0: stability check OK

---

## Post

| Block | Aksi |
|---|---|
| `success` | Print ringkasan deploy |
| `failure` | `helm rollback <service> 0` — rollback ke revision terakhir yang berhasil |
| `cleanup` | Hapus `service-src/` dan `devops-repo/` dari workspace |

---

## Perbedaan per Environment

| ENV | `PARAMETERS_BLOCK` | `TRIGGER_BLOCK` | `CHECKOUT_BRANCH` |
|---|---|---|---|
| `dev` | — | `githubPush()` | `dev` |
| `uat` | — | `githubPush()` | `uat` |
| `stag` | — | `githubPush()` | `main` |
| `prod` | `GIT_BRANCH` parameter | — | `${params.GIT_BRANCH}` |

`prod` tidak punya auto-trigger — deploy hanya bisa dijalankan manual dengan memilih branch atau tag yang ingin di-deploy.

---

## Credential yang Dibutuhkan

| Credential ID | Tipe | Dipakai di stage |
|---|---|---|
| `{{ GIT_CRED_ID }}` | SSH Private Key | Checkout |
| `{{ REGISTRY_CRED_ID }}` | Username/Password | — (via projected secret ke kaniko) |
| `{{ REGISTRY_SECRET }}` | Kubernetes Secret `docker-registry` | Build & Push Image |
| `{{ HELM_OCI_CRED_ID }}` | Username/Password | Prepare Helm |
| `{{ KUBECONFIG_CRED_ID }}` | File | Deploy, Post-Deploy Check |
| `{{ VAULT_CRED_ID }}` | Secret Text | Build & Push Image (khusus `vuejs`) |

`{{ REGISTRY_SECRET }}` harus ada di dua namespace:
- `jenkins` — untuk pull image `kaniko` dan `helm` saat pod dibuat
- `{{ ENV }}` — untuk pull image service saat Kubernetes menjalankan pod
