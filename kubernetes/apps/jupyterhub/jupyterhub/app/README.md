# JupyterHub

This directory deploys a JupyterHub instance into the `jupyterhub` namespace using the upstream Helm chart (`jupyterhub/jupyterhub`).

## What gets applied here (public repo)

- `helmrelease.yaml`: `HelmRepository` + `HelmRelease` for chart `jupyterhub` (pinned) and the baseline, non-sensitive values.
  - Supports private overlays via `valuesFrom` (ConfigMap `jupyterhub-values` and Secret `jupyterhub-secrets`, both optional).
  - Ingress is disabled; the instance is published with a HTTPRoute.
  - Hub database uses a PVC on storage class `zfs-configuration`.
  - User servers:
    - Default image: `quay.io/jupyter/scipy-notebook:python-3.13`
    - GPU profile: `quay.io/jupyter/pytorch-notebook:cuda12-python-3.13` with `runtimeClassName: nvidia` and `nvidia.com/gpu: "1"`
    - Requests/limits: CPU `2–8`, memory `8Gi–32Gi`
    - Static home storage mounted from `jupyterhub-users-pvc` (50Gi)
    - Extra tmpfs `/dev/shm` (4Gi) for notebooks that benefit from shared memory.
  - Idle culling enabled (1h timeout, 10m interval).
- `pvc.yaml`: `PersistentVolumeClaim` named `jupyterhub-users-pvc` (50Gi) that binds to a pre-created PV (`volumeName: jupyterhub-users-pv`) using storage class `zfs-configuration`.
- `kustomization.yaml`: wires the above into a single app unit for Flux.

## Secure counterpart (private repo)

Cluster-specific config and secrets are intentionally *not* stored here. The HelmRelease is designed to pull additional values from:

- ConfigMap `jupyterhub-values` (non-secret overrides)
- Secret `jupyterhub-secrets` (sensitive values; SOPS-encrypted at rest)
