# Routing

[← Docs index](../README.md)

By default, applications name a deployed model (for example `qwen2-5-coder-14b`). **Routing** uses a virtual model name instead. The stack classifies the prompt, selects a deployed model, and returns the completion. The client still calls `/v1/chat/completions`. `"model"` is the virtual name.

Routing is optional. A request that names a real model goes to that model. Use this page when one application-facing name should fan out to several backends.

| Requirement | Mechanism | Guide |
|---|---|---|
| Intent or category (code, chat, reasoning) | Semantic routing (`vllm-sr/auto`) | [Semantic routing](#semantic-routing) |
| Complexity, cost, and latency | Hybrid router (`hybrid-router`) | [Hybrid router](hybrid-router.md) |

Use one mechanism per virtual model name.

---

## Semantic routing

Semantic routing sits in front of models you have already deployed. A request with `"model": "vllm-sr/auto"` is classified and sent to one of those models. A request that names a real model is not routed.

- Coding prompts are sent to a coding model; general chat is sent to a general-purpose model.
- Lower-complexity traffic remains on a smaller, faster model; heavier prompts use a larger model.
- For families such as Qwen3 and DeepSeek, reasoning modes can be enabled when the prompt requires them.

The classifier and embedding models used by the router are cached on a persistent volume so a pod restart does not trigger a re-download.

### Enable semantic routing

Semantic routing is disabled by default. Enabling it does not alter models that applications already call by name.

1. In `env/<env>/config.inference.yaml` (the default environment is `local`):

   ```yaml
   vllm_sr_enabled: true
   ```

2. Deploy:

   ```bash
   ./toolkit_manager.sh install agent-toolkit
   ```

   To install only this piece on a stack that is already up: `./toolkit_manager.sh install vllm_sr --only`.

The installer is idempotent. Re-run it at any time; toolkit roles upgrade in place with the current YAML.

### Invoke the router

The virtual model name is `vllm-sr/auto` (`vllm_sr_litellm_model_name` in `config.yaml`).

With `auth_type: litellm_vkey`, call LiteLLM (default domain `inteltoolkits.ai`). Use a virtual key for apps; the master key is for install smoke tests only:

```bash
curl --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "litellm.${DOMAIN}:443:${GATEWAY_IP}" \
  "https://litellm.${DOMAIN}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${AGENT_TOOLKIT_API_KEY}" \
  -d '{
    "model": "vllm-sr/auto",
    "messages": [{"role": "user", "content": "write a hello world in python"}]
  }'
```

Set `DOMAIN` and `GATEWAY_IP` the same way as the [quickstart verify](../getting-started/quickstart.md#5-confirm-it-works). Guardrails and Langfuse apply the same as any other LiteLLM call. In this release the router API is not published on its own host. Send `vllm-sr/auto` through LiteLLM.

### Per-request selection

- `"model": "vllm-sr/auto"`: the router selects a deployed model.
- `"model": "qwen2-5-coder-14b"` (any deployed catalog name): that model is used. The router is bypassed.

### From Python or an orchestration framework

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://litellm.inteltoolkits.ai/v1",  # or litellm.<base_domain_name>
    api_key="<virtual-key>",  # not the LiteLLM master key
)

resp = client.chat.completions.create(
    model="vllm-sr/auto",
    messages=[{"role": "user", "content": "Refactor this function to run asynchronously"}],
)
print(resp.choices[0].message.content)
```

LangGraph, CrewAI, and other OpenAI-compatible clients use the same base URL and `model="vllm-sr/auto"`.

### Models eligible for routing

Models the router can select must already be deployed (`llm_services_autodeploy_models`, or `./model-manager deploy <name>`). Category mapping is `vllm_sr_config_override` in `config.inference.yaml`. Dashboard, signals, Helm settings, and troubleshooting: [vLLM semantic router](semantic-router.md).

Apply a change with:

```bash
./toolkit_manager.sh install agent-toolkit
```

### Verify routing

```bash
curl --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "litellm.${DOMAIN}:443:${GATEWAY_IP}" \
  "https://litellm.${DOMAIN}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${AGENT_TOOLKIT_API_KEY}" \
  -d '{"model": "vllm-sr/auto",
       "messages": [{"role": "user", "content": "Write a Python function to reverse a linked list"}]}'
```

Open `https://langfuse.<base_domain_name>` and inspect the `model` field on each trace.

---

## Next steps

- Complexity routing, on by default: [Hybrid router](hybrid-router.md)
- Which models install: [Choose a model](../getting-started/quickstart.md#3-choose-a-model)
- Model serving flags: [Configuration](configuration.md)
- Traces after a routed call: [backend connection reference](../reference/backend-connections.md) (Langfuse)
- Failures: [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md)

## Related pages

- [Meet the Agent Toolkit](../overview/meet.md)
- [Glossary](../reference/glossary.md#routing)
- [Guardrails](guardrails.md)
