# Airflow

This directory deploys an Apache Airflow instance into the `ml` namespace using the upstream Helm chart (`apache-airflow/airflow`).

## What gets applied here (public repo)

- `helmrelease.yaml`: `HelmRepository` + `HelmRelease` for chart `airflow` (pinned) and the baseline, non-sensitive values.
  - Deploys Airflow with `KubernetesExecutor`.
  - Uses an external PostgreSQL database instead of the chart-managed database.
  - Disables bundled Redis, Flower, PgBouncer, and chart ingress.
  - Applies the Flux-safe settings recommended by the upstream chart:
    - `createUserJob.useHelmHooks: false`
    - `createUserJob.applyCustomEnv: false`
    - `migrateDatabaseJob.useHelmHooks: false`
    - `migrateDatabaseJob.applyCustomEnv: false`
  - Enables git-sync for DAG delivery.
  - Uses an existing logs PVC (`airflow-logs-pvc`) instead of a chart-managed logs volume.
  - Supports private overlays via `valuesFrom`:
    - Secret `airflow-values`
- `db.yaml`: `CloudNativePG` `Cluster` named `airflow-db`.
  - PostgreSQL `17`
  - single instance
  - `10Gi` on storage class `nvmeof-fast`
  - bootstrapped from Secret `airflow-postgres-secret`
- `pvc.yaml`: `PersistentVolumeClaim` named `airflow-logs-pvc`.
  - `ReadWriteMany`
  - `2Gi`
  - statically bound to `airflow-logs-pv`
  - `storageClassName: zfs-configuration`
- `kustomization.yaml`: wires the above into a single app unit for Flux.

## Secure counterpart (private repo)

Cluster-specific config and secrets are intentionally *not* stored here. The private counterpart at `homelab-configuration/kubernetes/apps/ml/airflow/app` provides:

- `airflow-postgres-secret.sops.yaml`:
  - SOPS-encrypted PostgreSQL bootstrap credentials for `airflow-db`
- `airflow-values.sops.yaml`:
  - SOPS-encrypted Helm values passed through Secret `airflow-values`
  - includes the metadata DB connection, admin bootstrap user, webserver/JWT/Fernet secrets, base URL, and git-sync repo settings
- `pv.yaml`:
  - statically managed `PersistentVolume` for `airflow-logs-pv`
- `httproute.yaml`:
  - publishes Airflow at `airflow.${SECRET_DOMAIN}`
  - attaches to the existing Gateway
  - routes to service `airflow-api-server` on port `8080`

## Deployment shape

- Namespace: `ml`
- Chart: `apache-airflow/airflow`
- Executor: `KubernetesExecutor`
- Database: external CloudNativePG cluster (`airflow-db`)
- DAG delivery: git-sync
- Logs:
  - existing static PVC/PV pair
  - `airflow-logs-pvc` -> `airflow-logs-pv`
- Public access:
  - HTTPRoute in the private repo
  - chart ingress disabled
