# Single-node quickstart

[← Docs index](../README.md)

This walkthrough installs the Agent Toolkit on **one host**: a single machine, physical or virtual, running on Intel® Platforms. That host is the only Kubernetes node. When it finishes, you have a cluster, a model behind LiteLLM, and an OpenAI-compatible API you can call with `curl` or any OpenAI SDK.

Complete **[Prerequisites](prerequisites.md)** first (SSH and host prep). No certificate is needed: the installer generates one. `/etc/hosts` can wait until after install for browsers; the smoke test uses `--resolve`. If you have more than one host, use [multi-node deployment](../deployment/multi-node.md) instead.

Run every command from the [Toolkit Manager](https://github.com/intel/toolkit-manager) directory, the one that contains `toolkit_manager.sh`.

## 1. Install Toolkit Manager and seed config

```bash
git clone https://github.com/intel/toolkit-manager.git
cd toolkit-manager
./toolkit_manager.sh configure
./toolkit_manager.sh init agent-toolkit     # writes env/local/
```

`init` creates the YAML you edit next. It is not the install.

## 2. Edit the environment files

All live config is under `env/local/`. Change those files, not the copies in git.

### `env/local/global_config.yaml`: domain

```yaml
base_domain_name: inteltoolkits.ai   # Toolkit Manager default; change only if you use your own domain
```

| Field | What it means |
|---|---|
| `base_domain_name` | Hostname of the gateway (default `inteltoolkits.ai`). Must match DNS or `/etc/hosts`. |

Leave these two at their defaults:

| Field | Default | What it means |
|---|---|---|
| `gateway_tls_mode` | `selfsigned` | The installer creates a root CA and a certificate for `base_domain_name` and `*.base_domain_name`. To use your own certificate, see [Production TLS](prerequisites.md#production). |
| `storage_backend` | `local-path` | Disk on this node. Use `nfs` or `ceph` on multi-node. |

Already have a cluster? Set `existing_kubernetes: /path/to/kubeconfig` to skip building Kubernetes.

### `env/local/config.agent-toolkit.yaml`: leave the defaults

`init` already set `auth_type: litellm_vkey` and `llm_services_autodeploy_models: ["qwen2-5-coder-14b"]`, with Redis, pgvector, and the sandbox on. Leave that file alone for the first run.

## 3. Choose a model (only if you need to)

The default chat model is `qwen2-5-coder-14b` (about 96 GiB for the model pod; ~131 GiB for the whole stack). On a 128 GB node, edit `config.agent-toolkit.yaml`:

```yaml
llm_services_autodeploy_models: ["qwen2-5-coder-7b"]
```

Use a catalog name from `env/local/models.yaml`, not a raw Hugging Face id. To see every name `init` seeded:

```bash
./model-manager list --catalog
```

Deploy more later with `./model-manager deploy <name>`. Full steps: [Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md) in Toolkit Manager.

## 4. Install

```bash
./toolkit_manager.sh install agent-toolkit
```

This builds Kubernetes (unless you set `existing_kubernetes`), then model serving and the toolkit. Re-run the same command to resume a failed run. The first run is about 20–40 minutes; the model pod can take another 10–15 minutes to download weights. The default model is ungated, so no Hugging Face token is needed here. If you later pick a gated model (Llama, Gemma), export `HF_TOKEN` in the same shell before `install` or `./model-manager deploy` — see [Hugging Face token](../configuration/configuration.md#hugging-face-token).

## 5. Confirm it works

Same check as Toolkit Manager ([Agent Toolkit](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/agent-toolkit.md)): `--resolve` plus the installer CA. Browsers still need `/etc/hosts` (or DNS) after you have the gateway IP; see [Prerequisites](prerequisites.md#2-dns-and-tls).

```bash
export KUBECONFIG=$(pwd)/env/local/kubeconfig.yaml

kubectl get nodes
kubectl get pods -A

GATEWAY_IP=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg-gateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
DOMAIN=$(yq '.base_domain_name' env/local/global_config.yaml)
echo "$GATEWAY_IP  litellm.$DOMAIN"   # optional: also add to /etc/hosts for browsers

export LITELLM_MASTER_KEY=$(kubectl get secret litellm-master-key -n litellm \
  -o jsonpath='{.data.master_key}' | base64 -d)

curl --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "litellm.$DOMAIN:443:$GATEWAY_IP" \
  "https://litellm.$DOMAIN/v1/models" \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}"

curl --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "litellm.$DOMAIN:443:$GATEWAY_IP" \
  "https://litellm.$DOMAIN/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -d '{"model": "qwen2-5-coder-14b",
       "messages": [{"role": "user", "content": "Write a Python hello world"}],
       "max_tokens": 100}'
```

Use the `id` from `/v1/models` as `"model"`. For the default install that id is `qwen2-5-coder-14b`, not the Hugging Face repo name.

The master key lives in secret `litellm-master-key` (namespace `litellm`). Inside the LiteLLM pod it is `PROXY_MASTER_KEY`. The in-cluster service is `litellm.litellm.svc.cluster.local:4000`.

`--cacert` trusts the root CA the installer exports (`env/local/logs/toolkit-manager-ca.crt`). Use `-k` only if you want to skip verify. Production should use a CA-issued certificate.

## What also came up

You did not install these by hand. The install started them. Verify when you need them; URLs are in the [backend reference](../reference/backend-connections.md).

| Component | Check | In-cluster URL |
|---|---|---|
| **Redis** (session memory) | `kubectl get pods -n redis` then `kubectl exec -n redis redis-stack-server-0 -- sh -c 'redis-cli -a "$REDIS_PASSWORD" --no-auth-warning ping'` | Read `REDIS_URL` from secret `redis-stack-server-credentials` |
| **pgvector** (long-term memory) | `kubectl get pods -n pgvector` | Read `DATABASE_URL` from secret `pgvector-credentials` |
| **Agent sandbox** | `kubectl get pods -n agent-sandbox-system` and `kubectl get sandboxtemplate -n agent-sandbox-system` | `http://sandbox-router-svc.agent-sandbox-system.svc.cluster.local:8080` |

Redis requires a password (the chart will not render without one). The Service is ClusterIP: do not change it to NodePort. Full notes: [backend reference](../reference/backend-connections.md#redis). Sandbox templates and SDK: [Agent sandbox](../reference/agent-sandbox.md).

## Optional: route simple vs complex prompts

Send simple prompts to a smaller model and hard ones to a larger one without changing clients. This release creates a `hybrid-router` virtual model at install (`hybrid_router_enabled: true`). It is not required for the chat completion above. Full tier and bandit settings: [Hybrid router](../configuration/hybrid-router.md). Intent-based routing is in [Routing](../configuration/routing.md).

By default every tier points at the first entry in `llm_services_autodeploy_models` (`hybrid_router_tier_candidates`). Adding models to that list alone does not put them on the router. Set the tier lists yourself in `env/local/config.agent-toolkit.yaml` (or `config.inference.yaml`), deploy the models, then refresh LiteLLM with `litellm_reconcile` — an existing LiteLLM release keeps its old settings until you do:

```yaml
llm_services_autodeploy_models: ["qwen2-5-coder-14b", "bge-base-en", "qwen3-coder-30b"]
hybrid_router_default_model: qwen2-5-coder-14b
hybrid_router_tier_candidates:
  SIMPLE: ["qwen2-5-coder-14b"]
  MEDIUM: ["qwen2-5-coder-14b", "qwen3-coder-30b"]
  COMPLEX: ["qwen3-coder-30b"]
  REASONING: ["qwen3-coder-30b"]
```

```bash
./toolkit_manager.sh install agent-toolkit -- -e litellm_reconcile=true
```

Call `"model": "hybrid-router"`. Which real model answered shows in Langfuse at `https://langfuse.<base_domain_name>`.

## Next steps

Once a chat completion succeeds:

- Connect an agent: [Build your first agent](first-agent.md).
- Change domain, models, auth, or optional components: [Configuration](../configuration/configuration.md).
- In-cluster URLs: [backend connection reference](../reference/backend-connections.md).
- More: [Guardrails](../configuration/guardrails.md), [Routing](../configuration/routing.md), [multi-node](../deployment/multi-node.md), [decommission](../deployment/decommission.md), [FAQ](../faq-and-troubleshooting/frequently-asked-questions.md), [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md).
