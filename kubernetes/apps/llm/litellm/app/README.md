# LiteLLM client virtual keys

LiteLLM uses two different kinds of keys in this stack:

- `litellm-master-key` is the proxy administration key. It is wired into the Helm chart and should only be used for admin API operations.
- Client keys are LiteLLM virtual keys. They are stored as SOPS-backed Kubernetes Secrets and must also exist as database records inside LiteLLM.

The SOPS Secrets are the source of truth for client key material. The LiteLLM database is reconciled from those Secrets with:

```sh
kubectl -n llm port-forward svc/litellm 4000:4000
./scripts/litellm-sync-virtual-keys.sh
```

Set `LITELLM_URL` if the proxy is reachable somewhere other than `http://127.0.0.1:4000`.

By default the script reads key material from live Kubernetes Secrets, which is the normal GitOps path after Flux has applied the SOPS files. For bootstrap or recovery before Flux has applied the Secrets, run it from the private `homelab-configuration` repo on a machine with the SOPS age identity:

```sh
LITELLM_KEY_SOURCE=sops ./scripts/litellm-sync-virtual-keys.sh
```

## Current keys

| Alias | Secret | Namespace | Models |
| --- | --- | --- | --- |
| `openwebui` | `openwebui-litellm-api-key` | `ollama` | All current LiteLLM aliases |
| `continue` | `litellm-continue-api-key` | `llm` | All current LiteLLM aliases |

There is intentionally no Gatus LiteLLM key. The current generated Gatus check for LiteLLM is a guarded DNS check and does not call the LiteLLM API.

There is intentionally no Codex key in this repo because there is no repo-managed Codex LiteLLM/OpenAI config.

## Why the sync script exists

Kubernetes Secrets make the key string available to clients such as OpenWebUI and Continue. LiteLLM still validates virtual keys against database records that contain the allowed models, alias, metadata, and blocked state.

The sync script bridges those two systems:

1. Reads the SOPS-backed Kubernetes Secrets, or decrypts the local SOPS files when `LITELLM_KEY_SOURCE=sops`.
2. Authenticates to LiteLLM with `litellm-master-key`.
3. Updates existing virtual-key records, or creates them if missing.
4. Can be rerun safely after a LiteLLM database restore or rebuild.

## Rotation

1. Generate a new key value:

   ```sh
   printf 'sk-%s\n' "$(openssl rand -hex 32)"
   ```

2. Store it with `sops edit` in the relevant private-repo Secret:
   - OpenWebUI: `kubernetes/apps/ollama/openwebui/app/openwebui-litellm-api-key.sops.yaml`
   - Continue: `kubernetes/apps/llm/litellm/app/litellm-continue-api-key.sops.yaml`

3. Let Flux apply the Secret.
4. Port-forward LiteLLM and run `./scripts/litellm-sync-virtual-keys.sh`.
5. Restart or reload the affected client if needed.
6. Block the old key from LiteLLM only after the new key validates.

## Validation

With a port-forward running:

```sh
OPENWEBUI_KEY="$(kubectl -n ollama get secret openwebui-litellm-api-key -o jsonpath='{.data.api-key}' | base64 -d)"
CONTINUE_KEY="$(kubectl -n llm get secret litellm-continue-api-key -o jsonpath='{.data.api-key}' | base64 -d)"
MASTER_KEY="$(kubectl -n llm get secret litellm-master-key -o jsonpath='{.data.masterkey}' | base64 -d)"

curl -fsS http://127.0.0.1:4000/v1/models -H "Authorization: Bearer ${OPENWEBUI_KEY}"
curl -fsS http://127.0.0.1:4000/v1/models -H "Authorization: Bearer ${CONTINUE_KEY}"

[ "${MASTER_KEY}" != "${OPENWEBUI_KEY}" ]
[ "${MASTER_KEY}" != "${CONTINUE_KEY}" ]
```

OpenWebUI keeps its existing Ollama configuration through `ollamaUrls`; this virtual-key setup only changes the LiteLLM/OpenAI-compatible key.
