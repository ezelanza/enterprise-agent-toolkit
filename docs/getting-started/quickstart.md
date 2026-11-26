# Single-node quickstart

[← Docs index](../README.md)

This walkthrough installs the Agent Toolkit on **one host**: a single machine, physical or virtual, running on Intel® Platforms. That host is the only Kubernetes node. When it finishes, you have a cluster, a model behind LiteLLM, and an OpenAI-compatible API you can call with `curl` or any OpenAI SDK.

Complete **[Prerequisites](prerequisites.md)** first so SSH and `/etc/hosts` (or DNS) are already in place. No certificate is needed: the installer generates one. If you have more than one host, stop here and use [multi-node deployment](../deployment/multi-node.md) instead: the YAML is the same, but the inventory is not.

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

### `env/local/config.agent-toolkit.yaml`: auth, model, and components

`init` writes `auth_type: litellm_vkey` so the LiteLLM gateway, Langfuse, and the content-safety guardrail come up together. Leave that for the first run. For Keycloak JWT on the AI gateway instead, set `auth_type: jwt` in this file. The old `auth_provider` key in `global_config.yaml` is deprecated; use `auth_type` only.

The model list is `llm_services_autodeploy_models`. The default is `qwen2-5-coder-14b`. Redis, pgvector, and the sandbox are on. KubeRay is off:

```yaml
auth_type: litellm_vkey
llm_services_autodeploy_models: ["qwen2-5-coder-14b"]
redis_stack_enabled: true
pgvector_enabled: true
agent_sandbox_enabled: true
kuberay_enabled: false
openshell_enabled: false
```

Set `kuberay_enabled: true` only if you need Ray. The full guide is [KubeRay](../reference/kuberay.md). Other models are deployed later with `./model-manager deploy <name>`.

### Hugging Face token

Required before install when a model in that list is gated (Llama, Gemma). The default `qwen2-5-coder-14b` is not gated.

```bash
export HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx
```

## 3. Choose a model

Set `llm_services_autodeploy_models` to one or more names from this table. Do not invent a Hugging Face id here. The chat `"model"` value is this name, for example `qwen2-5-coder-14b`.

| Value | Model |
|---|---|
| `qwen2-5-coder-14b` | Qwen/Qwen2.5-Coder-14B-Instruct. Default |
| `qwen2-5-coder-7b` | Qwen/Qwen2.5-Coder-7B-Instruct. Fits a 128 GB node |
| `qwen3-coder-30b` | Qwen/Qwen3-Coder-30B-A3B-Instruct |
| `qwen3-30b-a3b` | Qwen/Qwen3-30B-A3B-Instruct-2507 |
| `qwen3-6-35b-a3b` | Qwen/Qwen3.6-35B-A3B |
| `gemma-4-26b` | google/gemma-4-26B-A4B-it (gated) |
| `llama-3-1-8b` | meta-llama/Llama-3.1-8B-Instruct (gated). Sequential tool calls only |
| `llama-3-1-8b-specdec` | The same model with speculative decoding |
| `qwen3-coder-30b-specdec` | The 30B coder with speculative decoding |
| `qwen2-5-coder-14b-specdec` | The default coder with speculative decoding |
| `bge-base-en` | BAAI/bge-base-en-v1.5 (embeddings) |
| `bge-reranker` | BAAI/bge-reranker-base (reranking) |

A `-specdec` name deploys that model with a smaller draft model. The draft is configured for you. The chat `"model"` value keeps the suffix, for example `qwen3-coder-30b-specdec`.

The default model requests about 96 GiB, and the rest of a default install (guard model, router, platform) brings the total to about 131 GiB. On a 128 GB node, use `qwen2-5-coder-7b` or `llama-3-1-8b` instead. Per-model CPU and memory live in `env/local/models.yaml`. To add or size a model, see [Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md) in Toolkit Manager.

On a node with one NUMA node, `cpu: auto` on the default model can leave no cores for a second model. `./model-manager deploy` then stops with `NRI pre-flight failed: not enough physical cores`. Set an explicit `cpu:` for the first model, or undeploy it with `./model-manager undeploy <name>` before deploying another. `model-manager` is in the Toolkit Manager checkout, next to `toolkit_manager.sh`.

## 4. Install

```bash
export HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx   # omit when every selected model is ungated
./toolkit_manager.sh install agent-toolkit
```

This builds Kubernetes (unless you brought your own cluster with `existing_kubernetes`), then model serving and the toolkit. Re-run the same command to resume a failed run. The first run is about 20–40 minutes; the model pod can take another 10–15 minutes to download weights.

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

Once a chat completion succeeds, most people either change which models are served or point an application at the in-cluster backends:

- Build a LangChain agent on the gateway and trace it in Langfuse: [Build your first agent](first-agent.md).
- Edit the YAML under `env/local/` using [Configuration](../configuration/configuration.md).
- Copy Redis, pgvector, and sandbox URLs from the [backend connection reference](../reference/backend-connections.md).
- Tune or turn off content-safety screening in [Guardrails](../configuration/guardrails.md), or turn on Ray jobs in [KubeRay](../reference/kuberay.md).
- Add another server with [multi-node deployment](../deployment/multi-node.md), or [tear the stack down](../deployment/decommission.md) and install again.
- Optional routing choices are in [Routing](../configuration/routing.md). Common questions: [FAQ](../faq-and-troubleshooting/frequently-asked-questions.md). Terms: [glossary](../reference/glossary.md). Failures: [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md).
