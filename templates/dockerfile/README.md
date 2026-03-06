# Dockerfile Templates

Template ini dibaca oleh `Jenkinsfile.generator` saat generate service baru.
Lokasi hasil generate: `services/<service>/<env>/Dockerfile`

## Struktur

```
templates/dockerfile/
├── Dockerfile.vuejs.template
├── Dockerfile.golang.template
├── Dockerfile.node-npm.template
├── Dockerfile.python-pip.template
├── Dockerfile.static-nginx.template
├── Dockerfile.node-yarn.template      # coming soon
├── Dockerfile.java-jar.template       # coming soon
└── Dockerfile.custom-cmd.template     # coming soon
```

## Placeholder

| Placeholder | Diisi oleh | Contoh |
|---|---|---|
| `{{ SERVICE }}` | Generator — nama service | `ghost` |
| `{{ ENV_UPPER }}` | Generator — environment uppercase | `UAT` |
| `{{ APP_PORT }}` | Generator — dari parameter `CONTAINER_PORT` | `3000` |
| `{{ VAULT_ADDR }}` | Generator — dari parameter `VAULT_ADDR` | `https://vault.example.com` |
| `{{ VAULT_SECRET_PATH }}` | Generator — dari parameter `VAULT_SECRET_PATH` | `secret/data/ghost/uat` |

## Default Base Image per Runtime

| Runtime | Builder image | Runtime image |
|---|---|---|
| `vuejs` | `node:20-alpine` | `node:20-alpine` |
| `golang` | `golang:1.22-alpine` | `gcr.io/distroless/static-debian12` |
| `node-npm` | `node:20-alpine` | `node:20-alpine` |
| `python-pip` | `python:3.12-slim` | `python:3.12-slim` |
| `static-nginx` | `node:20-alpine` | `nginx:1.27-alpine` |

Builder image dan runtime image hardcode di masing-masing file template.
Edit file `.template` langsung jika perlu ganti versi image.

## OWASP Standards yang Diterapkan

| Kode | Kontrol | Template |
|---|---|---|
| DS-01 | Multi-stage build | Semua |
| DS-02 | Non-root user (UID 10001 / nginx UID 101 / nonroot UID 65532) | Semua |
| DS-03 | Minimal runtime image (distroless / alpine) | golang, vuejs, static-nginx |
| DS-06 | HEALTHCHECK built-in | Semua |
| DS-07 | `COPY --chown` | Semua |
| DS-08 | Pinned image tag | Semua |
| DS-09 | `ARG VAULT_TOKEN` tidak persist ke runtime layer | vuejs |

## Cara Edit Template

Edit file `.template` langsung di repo ini, lalu jalankan ulang generator.
Generator akan membaca versi terbaru dari template saat pipeline dijalankan.

## Catatan vuejs

`.env` hasil fetch dari Vault di-bake ke dalam image.
Pastikan image hanya di-push ke **private registry** (Harbor / Huawei SWR).
Jangan push ke Docker Hub public.
