# homelab-gitops

Public, reusable GitOps manifests for Kubernetes apps in my homelab.

This repository intentionally contains **only the “pretty bits”**: application-layer Kubernetes manifests and Flux-compatible structure.
It **does not** contain cluster bootstrap, Talos configuration, networking/router automation, storage/NFS plumbing, internal DNS/IPs, or secrets.

The actual cluster is bootstrapped from a private repository which:
- sets up the router VM and networking bits with Ansible
- sets up Talos VMs (if needed) with Ansible
- installs/bootstraps Talos
- installs/bootstraps Flux
- defines secrets (SOPS) and private overlays
- wires this repository in as a **secondary** source via Flux `GitRepository` + `Kustomization`

If you’re browsing this repo: everything here is meant to be safe to publish and reasonably reusable.


## What lives here

- Kubernetes application manifests organized in a Flux-friendly tree
- Helm-based apps expressed as:
  - `HelmRepository`
  - `HelmRelease`
  - app-local `kustomization.yaml`
