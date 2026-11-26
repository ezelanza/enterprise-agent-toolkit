# Hybrid router

[← Docs index](../README.md)

Call one name, `"model": "hybrid-router"`, and let the stack send simple prompts to a smaller model and hard ones to a larger model. This release creates that virtual model at install (`hybrid_router_enabled: true` in `config.agent-toolkit.yaml`). It runs with `auth_type: litellm_vkey`. Set tier lists in `hybrid_router_tier_candidates` (see the [quickstart optional section](../getting-started/quickstart.md#optional-route-simple-vs-complex-prompts)). Intent-based routing is in [Routing](routing.md).

```bash
curl --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "litellm.${DOMAIN}:443:${GATEWAY_IP}" \
  "https://litellm.${DOMAIN}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${AGENT_TOOLKIT_API_KEY}" \
  -d '{"model": "hybrid-router",
       "messages": [{"role": "user", "content": "Write a Python hello world"}],
       "max_tokens": 100}'
```

Use a virtual key for apps. Which real model answered shows in Langfuse. The sections below cover classification, bandit settings, and how the router mounts into LiteLLM.

## How it works

At proxy startup, the module `custom_hybrid_router.py` patches `litellm.router.Router` to recognize the `auto_router/hybrid_router` deployment prefix. This mirrors LiteLLM's built-in `ComplexityRouter`. A pre-routing hook then handles every request to the virtual model before LiteLLM chooses a deployment:

```
Request model="hybrid-router"
  │
  ▼
Router.async_pre_routing_hook (patched by custom_hybrid_router.py)
  │  model in hybrid_routers?  NO  → normal LiteLLM routing
  │                            YES ↓
  │  1. Extract user and system text from messages
  │  2. HybridRouter.classify()   → SIMPLE / MEDIUM / COMPLEX / REASONING
  │  3. HybridRouter.pick_model() → best real model for that tier (bandit)
  │  4. Return PreRoutingHookResponse(model=<chosen real model>)
  │
  ▼
LiteLLM routes to the chosen model
  │
  ▼
async_log_success_event / async_log_failure_event
  Latency and token throughput → reward → bandit.update()
```

Requests to any other model name are not affected. The `auto_router/hybrid_router` model is never dispatched directly.

### Layer 1: complexity classification

A rules-based classifier scores each prompt without an extra model call. It weighs estimated token count, code, reasoning, technical, and simple keyword hits, multi-step patterns (`first … then`, `step 2`, numbered lists), and question density, then maps the score to a tier:

| Tier | Score | Typical prompts |
|---|---|---|
| `SIMPLE` | `< 0.15` | Greetings, factual lookups, yes/no |
| `MEDIUM` | `0.15`–`0.35` | Explanations, summaries, basic coding |
| `COMPLEX` | `0.35`–`0.60` | Architecture, multi-step code, technical analysis |
| `REASONING` | `≥ 0.60`, or 2+ reasoning markers | Chain-of-thought, deduction, proofs |

Tier boundaries, token thresholds, keyword lists, and dimension weights can be overridden with `complexity_router_config`. The tier logic comes from LiteLLM's built-in `complexity_router` module; it is not vendored.

### Layer 2: efficiency (multi-armed bandit)

Each tier has its own bandit, whose arms are that tier's candidate models. Per-tier bandits matter because a model can be fast on `SIMPLE` prompts and slow on `COMPLEX` ones. After each response the router updates the arm with a reward:

```
time_per_token = latency_seconds / max(completion_tokens, 1)
reward         = target_tpt / (target_tpt + time_per_token)     # in (0, 1]
```

Failures record a reward of `0`. When `quality_weight > 0`, the reward is `(1 - quality_weight) * efficiency + quality_weight * quality_score`. A tier with only one candidate skips the bandit and routes directly.

| `bandit` | Algorithm | Behavior |
|---|---|---|
| `thompson` (default) | Beta-Bernoulli Thompson Sampling | Posterior sampling; balances exploration and exploitation |
| `ucb` | Upper Confidence Bound | Optimism under uncertainty; `delta` sets the exploration bound |
| `epsilon_greedy` | ε-greedy | Explores uniformly with probability `epsilon`, otherwise greedy |

## 1. Enable the router

Set the toggle in `env/local/config.inference.yaml`:

```yaml
hybrid_router_enabled: true
```

Then install Intel® Inference Toolkit (or the whole stack):

```bash
./toolkit_manager.sh install inference      # or: install agent-toolkit
```

With the toggle on, the `litellm` role (`tasks/hybrid_router_setup.yaml`):

1. Renders the router source into the `litellm-router-code` ConfigMap.
2. Mounts each Python file next to `config.yaml` in the LiteLLM container (`/etc/litellm/…`, where LiteLLM resolves `callbacks` modules) and sets `PYTHONPATH`.
3. Adds the callback, so LiteLLM imports the module and applies the patch at startup:

   ```yaml
   litellm_settings:
     callbacks: ["langfuse", "custom_hybrid_router.proxy_handler_instance"]
   ```

4. Registers the `hybrid-router` virtual model from the `hybrid_router_*` settings below.

A fresh install picks the router up directly. On a cluster where LiteLLM is already running, change the `hybrid_router_*` settings and re-run with `litellm_reconcile` so the release picks them up:

```bash
./toolkit_manager.sh install agent-toolkit -- -e litellm_reconcile=true
```

(or set `litellm_reconcile: true` in `config.agent-toolkit.yaml` / `config.inference.yaml`).

## 2. Configure tiers and the bandit

The `litellm` role builds the virtual model from these keys in `env/local/config.inference.yaml` and maps them onto `litellm_params.hybrid_router_config`:

| Field | Default | What it means |
|---|---|---|
| `hybrid_router_model_name` | `hybrid-router` | Model name clients call |
| `hybrid_router_default_model` | Required when `tier_candidates` is omitted. The install example uses `qwen2-5-coder-14b` | Fallback model |
| `hybrid_router_tier_candidates` | Default model for every tier | `tier_candidates`: tier → list of deployed model names |
| `hybrid_router_bandit` / `_delta` / `_epsilon` / `_target_tpt` / `_quality_weight` | See next table | Bandit parameters |
| `hybrid_router_tier_boundaries` | `0.15`, `0.35`, `0.60` | `complexity_router_config.tier_boundaries` |

Every model in `tier_candidates` must already be deployed (`llm_services_autodeploy_models`, or `./model-manager deploy <name>`).

The resulting `hybrid_router_config` keys:

| Key | Default | What it means |
|---|---|---|
| `bandit` | `thompson` | Algorithm: `ucb`, `epsilon_greedy`, or `thompson` |
| `delta` | `0.1` | UCB exploration; smaller means more exploration. Must be in `(0, 1)`. |
| `epsilon` | `0.1` | ε-greedy exploration probability. Must be in `[0, 1]`. |
| `target_tpt` | `1.0` | Target time per token; the midpoint of the reward curve |
| `quality_weight` | `0.0` | Quality vs. efficiency blend in `[0, 1]`; `0` is pure efficiency |
| `tier_candidates` | `hybrid_router_default_model` for every tier | Tier → list of real model names |
| `tier_priors` | None | Optional per-tier, per-arm Beta priors `(alpha, beta)` for Thompson Sampling |
| `complexity_router_config` | Built-in defaults | Tier boundaries, token thresholds, keyword lists, dimension weights |
| `hybrid_router_default_model` | None | Fallback model. Required if `tier_candidates` is omitted. |

`complexity_router_config.tier_boundaries` accepts `simple_medium`, `medium_complex`, and `complex_reasoning` (defaults `0.15`, `0.35`, `0.60`).

For reference, the role generates a LiteLLM entry equivalent to this one:

```yaml
model_list:
  - model_name: llama-8b
    litellm_params:
      model: openai/Llama-3.1-8B
      api_base: http://inference-8b:8000/v1
      api_key: not-needed
  - model_name: llama-70b
    litellm_params:
      model: openai/Llama-3.1-70B
      api_base: http://inference-70b:8000/v1
      api_key: not-needed

  # Virtual entry that clients call
  - model_name: hybrid-router
    litellm_params:
      model: auto_router/hybrid_router
      hybrid_router_default_model: llama-8b
      hybrid_router_config:
        bandit: thompson
        target_tpt: 1.0
        quality_weight: 0.0
        tier_candidates:
          SIMPLE:    [llama-8b, llama-70b]
          MEDIUM:    [llama-8b, llama-70b]
          COMPLEX:   [llama-70b]
          REASONING: [llama-70b]
        complexity_router_config:
          tier_boundaries:
            simple_medium: 0.15
            medium_complex: 0.35
            complex_reasoning: 0.60
```

## 3. Optional: add a virtual model at runtime

If the gateway is running with `store_model_in_db` enabled, you can add another virtual entry without editing config or restarting, through `/model/new`:

```bash
export LITELLM_MASTER_KEY=$(kubectl get secret litellm-master-key -n litellm \
  -o jsonpath='{.data.master_key}' | base64 -d)

# Or use the in-cluster endpoint http://litellm.litellm.svc.cluster.local:4000
curl -k https://litellm.<base_domain_name>/model/new \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "hybrid_router2",
    "litellm_params": {
      "model": "auto_router/hybrid_router",
      "hybrid_router_default_model": "Qwen/Qwen2.5-Coder-14B-Instruct",
      "hybrid_router_config": {
        "bandit": "thompson",
        "delta": 0.1,
        "epsilon": 0.1,
        "target_tpt": 1.0,
        "quality_weight": 0.0,
        "tier_candidates": {
          "SIMPLE":    ["meta-llama/Llama-3.1-8B-Instruct", "Qwen/Qwen2.5-Coder-14B-Instruct"],
          "MEDIUM":    ["meta-llama/Llama-3.1-8B-Instruct", "Qwen/Qwen2.5-Coder-14B-Instruct"],
          "COMPLEX":   ["Qwen/Qwen3-Coder-30B-A3B-Instruct"],
          "REASONING": ["Qwen/Qwen3-Coder-30B-A3B-Instruct"]
        },
        "complexity_router_config": {
          "tier_boundaries": {"simple_medium": 0.15, "medium_complex": 0.35, "complex_reasoning": 0.60},
          "token_thresholds": {"simple": 15, "complex": 400}
        }
      }
    },
    "model_info": {}
  }'
```

`model_name` is the name clients call (here `hybrid_router2`). Every model in `tier_candidates` and `hybrid_router_default_model` must already be onboarded on the gateway.

## 4. Test

```bash
curl -k https://litellm.<base_domain_name>/v1/chat/completions \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model": "hybrid-router", "messages": [{"role": "user", "content": "What is 2+2?"}]}'
```

The router classifies the prompt and forwards it. Requests to any other model name are unaffected.

## 5. Observe routing decisions

The proxy logs one line at startup and one per routed request:

```bash
kubectl logs -n litellm deploy/litellm -c litellm -f
```

```
[HybridRouterProxy] litellm.router.Router patched for auto_router/hybrid_router
[HybridRouterProxy] Initialised model_name=hybrid-router bandit=thompson default=llama-8b tiers=['SIMPLE', 'MEDIUM', 'COMPLEX', 'REASONING']
[HybridRouterProxy] model=hybrid-router call_id=... tier=SIMPLE -> llama-8b
```

Per-tier bandit state (arm counts, running means, and the Beta `alpha`/`beta` posteriors for Thompson) is available from `HybridRouterProxy.router_state()`. Latency and token metrics also reach Langfuse and Prometheus through the standard gateway telemetry.

Langfuse records which model answered. Open `https://langfuse.<base_domain_name>`. The login is in Secret `langfuse-secrets` in namespace `langfuse`:

```bash
kubectl get secret langfuse-secrets -n langfuse -o jsonpath='{.data.admin_email}' | base64 -d; echo
kubectl get secret langfuse-secrets -n langfuse -o jsonpath='{.data.admin_password}' | base64 -d; echo
```

Open the `inference` project. Under **Tracing**, select the latest `litellm_request`, then **Metadata** → **attributes**, and read `llm.model_name`.

### LiteLLM UI

The UI is `https://litellm.<base_domain_name>/ui`. Browsers need that name in DNS or `/etc/hosts` (smoke-test curls can use `--resolve` instead; see [Prerequisites](../getting-started/prerequisites.md#2-dns-and-tls)).

The username is `admin`. The password is `LITELLM_MASTER_KEY`. Models deployed with `./toolkit_manager.sh install` are listed under **Models+Endpoints** → **All Models**, including `bge-base-en` when that catalog name is in `llm_services_autodeploy_models`.

This release creates `hybrid-router` at install (`hybrid_router_enabled: true`). There is no UI step to create it. New models join a tier only when you list them in `hybrid_router_tier_candidates` and re-run install with `litellm_reconcile=true` (defaults keep every tier on the default model alone).

If a second model fails with `NRI pre-flight failed: not enough physical cores`, set an explicit `cpu:` in `models.yaml` ([Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md)), undeploy the first model, and install again:

```bash
kubectl get llminferenceservices,inferenceservices -n llm-inference
./model-manager undeploy <first_model_name>
./toolkit_manager.sh install agent-toolkit
kubectl rollout restart deployment litellm -n litellm
```

## Disable the router

Set `hybrid_router_enabled: false` in `env/local/config.inference.yaml`. On a fresh install, the ConfigMap, mounts, callback, and virtual model are not created. Because LiteLLM is install-once, removing them from a running release takes `helm uninstall litellm -n litellm` followed by `./toolkit_manager.sh install inference`. With the router off, LiteLLM behaves as a stock proxy.

## File layout

The router lives in the inference repository:

```
ai-inference/roles/litellm/
├── defaults/main.yaml                ← hybrid_router_* settings
├── config.yaml                       ← seeds config.inference.yaml (the toggle)
├── tasks/hybrid_router_setup.yaml    ← ConfigMap, mounts, callback, model wiring
└── files/router/
    ├── custom_hybrid_router.py       ← callback and Router patch (entry point)
    └── router_strategy/
        ├── efficiency_router/
        │   ├── bandit.py             ← UCB, EpsilonGreedy, ThompsonSampling
        │   └── efficiency_router.py  ← standalone efficiency router and reward functions
        └── hybrid_router/
            └── hybrid_router.py      ← two-layer router (classify, then pick)
```

## Next steps

- [Routing](routing.md): the other routing options (Auto Router, semantic routing)
- [Configuration](configuration.md): `hybrid_router_enabled` in `config.inference.yaml`
- [Quickstart: Auto Router](../getting-started/quickstart.md#optional-route-simple-vs-complex-prompts): LiteLLM UI routing that does not use this flag
- [Backend connection reference](../reference/backend-connections.md): Langfuse traces
- [Glossary](../reference/glossary.md#routing)
