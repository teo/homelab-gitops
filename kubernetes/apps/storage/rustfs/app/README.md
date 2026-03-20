# RustFS

This directory deploys a RustFS instance into the `storage` namespace using the upstream Helm chart (`rustfs/rustfs`).

## What gets applied here (public repo)

- `helmrelease.yaml`: `HelmRepository` + `HelmRelease` for chart `rustfs` (pinned) and the baseline, non-sensitive values.
  - Deploys RustFS in `standalone` mode.
  - Uses existing PVCs instead of chart-managed storage:
    - `rustfs-data-pvc` for object data
    - `rustfs-logs-pvc` for logs
  - Ingress is disabled.
  - Chart-managed Gateway API resources are also disabled; publication is handled with manually-authored `HTTPRoute` resources in the private repo.
  - Service type is `ClusterIP`.
  - Credentials come from an existing Secret (`rustfs-keys`) rather than chart-generated defaults.
  - Supports private overlays via `valuesFrom`:
    - ConfigMap `rustfs-values`
    - Secret `rustfs-secrets`
- `pvc.yaml`: PVCs used by the standalone deployment.
  - `rustfs-data-pvc`:
    - `ReadWriteOnce`
    - `100Gi`
    - `storageClassName: nvmeof-fast`
    - dynamically provisioned
  - `rustfs-logs-pvc`:
    - `ReadWriteOnce`
    - `256Mi`
    - statically bound to `rustfs-logs-pv`
    - `storageClassName: zfs-configuration`
- `kustomization.yaml`: wires the above into a single app unit for Flux.

## Secure counterpart (private repo)

Cluster-specific config and secrets are intentionally *not* stored here. The private counterpart at `homelab-configuration/kubernetes/apps/storage/rustfs/app` provides:

- `pv.yaml`:
  - the statically managed `PersistentVolume` for `rustfs-logs-pv`
- `httproute.yaml`:
  - `rustfs-s3` publishing the S3 endpoint
  - `rustfs-api` publishing the admin/API endpoint
  - both routes attach to the existing Gateway in `kube-system`
- `rustfs-keys.sops.yaml`:
  - SOPS-encrypted credentials used by `secret.existingSecret: rustfs-keys`

## Deployment shape

- Namespace: `storage`
- Chart: `rustfs`
- Mode: standalone, single-node
- Public access:
  - S3 endpoint via Gateway API route in the private repo
  - API/console endpoint via Gateway API route in the private repo
- Storage model:
  - data on dynamic `nvmeof-fast`
  - logs on static `zfs-configuration`
