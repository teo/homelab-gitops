# MLflow

This directory deploys an MLflow instance into the `ml` namespace using the upstream Helm chart (`community-charts/mlflow`).

## What gets applied here (public repo)

- `helmrelease.yaml`: `HelmRepository` + `HelmRelease` for chart `mlflow` (pinned) and the baseline, non-sensitive values.
  - Uses a private `ConfigMap` (`mlflow-values`) for cluster-specific configuration such as database host/credentials wiring and HTTP settings.
  - Runs with `strategy.type: Recreate`.
  - Mounts a dedicated PVC (`mlflow-data-pvc`) at `/mlflow/data`.
  - Uses PostgreSQL as the backend store.
  - Stores artifacts on the mounted local volume:
    - `/mlflow/data/mlruns`
    - `/mlflow/data/mlartifacts`
  - Service name is `mlflow`.
- `db.yaml`: `CloudNativePG` `Cluster` named `mlflow-db`.
  - PostgreSQL `17`
  - single instance
  - `10Gi` on storage class `nvmeof-fast`
  - bootstrapped from Secret `postgres-secret`
- `pvc.yaml`: `PersistentVolumeClaim` named `mlflow-data-pvc`.
  - `ReadWriteOnce`
  - `100Gi`
  - statically bound to `mlflow-data-pv`
  - `storageClassName: zfs-configuration`
- `kustomization.yaml`: wires the above into a single app unit for Flux.

## Secure counterpart (private repo)

Cluster-specific config and secrets are intentionally *not* stored here. The private counterpart at `homelab-configuration/kubernetes/apps/ml/mlflow/app` provides:

- `pv.yaml`:
  - the statically managed `PersistentVolume` for `mlflow-data-pv`
- `values.yaml`:
  - `ConfigMap` named `mlflow-values`
  - sets `allowed-hosts`, CORS origins, and PostgreSQL connection details
  - points MLflow at the CloudNativePG read-write service (`mlflow-db-rw.ml.svc.cluster.local`)
- `postgres-secret.sops.yaml`:
  - SOPS-encrypted PostgreSQL bootstrap credentials for `mlflow-db`
- `httproute.yaml`:
  - publishes MLflow at `mlflow.${SECRET_DOMAIN}`
  - attaches to the existing Gateway
  - routes to service `mlflow` on port `80`

## Deployment shape

- Namespace: `ml`
- Chart: `community-charts/mlflow`
- Database: external CloudNativePG cluster (`mlflow-db`)
- Artifacts:
  - local PVC-backed storage
  - `mlflow-data-pvc` -> `mlflow-data-pv`
- Public access:
  - HTTPRoute in the private repo
  - chart ingress is not used
