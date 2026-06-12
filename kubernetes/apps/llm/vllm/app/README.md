# vLLM

This directory deploys one local coding model through the official vLLM
Production Stack Helm chart.

## Deployment

- Namespace: `llm`
- One internal router and one serving engine
- One NVIDIA GPU assigned exclusively to the engine
- Model: `nvidia/Qwen3.6-35B-A3B-NVFP4`
- Cache: shared RWX model cache on `nfs-ssd`
- Route: `https://vllm.${SECRET_DOMAIN}` through the internal Gateway
- Monitoring: chart-managed router and engine `ServiceMonitor` resources

The engine uses `Recreate` because a rolling update cannot schedule a second
pod when only one GPU is available.

## Required secret

The private configuration repo provides SOPS-encrypted Secret `vllm-api-key`
in namespace `llm` with key `api-key`. The private `llm` app group also owns
the namespace, static cache PV, and common SOPS resources used by Flux.

The selected model is public and does not require a Hugging Face token. If a
future gated model needs one, set that model's `hf_token.secretName` and
`hf_token.secretKey` values to reference a separate SOPS-managed Secret.

## Storage and monitoring

The private configuration repo owns the retained NFS PV and its
`vllm-model-cache-claim` PVC. The chart mounts that existing `ReadWriteMany`
claim as `HF_HOME` for the engine, so Helm install remediation cannot delete
the model cache. Downloaded models, tokenizers, and Hugging Face cache data can
be reused by future vLLM workloads. The backing NFS directory must exist before
reconciliation. Configuration remains in the HelmRelease and Kubernetes
Secrets.

The chart's bundled Prometheus stack is disabled. The generated
`ServiceMonitor` resources are discovered by the cluster's existing
Prometheus. The chart's Grafana option emits dashboard ConfigMaps for a
sidecar-based Grafana installation, while this cluster uses the Grafana
Operator, so those ConfigMaps are intentionally not created.

## Initial sizing

The model is an NVIDIA NVFP4 checkpoint intended for vLLM on Blackwell GPUs.
Initial limits are deliberately conservative:

- tensor parallel size: `1`
- GPU memory utilization: `0.90`
- maximum model length: `32768`
- maximum concurrent sequences: `2`
- LMCache, LoRA adapters, KEDA, and autoscaling: disabled

LoRA support can be enabled with an adapter deployment later. KEDA remains out
of this baseline until scaling behavior and metrics have been validated.
