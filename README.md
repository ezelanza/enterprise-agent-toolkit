<img src="docs/assets/banner.jpg" alt="Intel® Agent Toolkit" width="100%">

<!-- vale off (due to hyperlinks) -->
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/intel/agent-toolkit/blob/main/LICENSE)
[![Platform: Intel® Xeon® processors](https://img.shields.io/badge/Platform-Intel%C2%AE%20Xeon%C2%AE%20processors-0068B5)](https://www.intel.com/xeon)
[![Platform: GPU](https://img.shields.io/badge/Platform-GPU-0068B5)](https://www.intel.com/content/www/us/en/products/details/discrete-gpus.html)
[![Deployment: Kubernetes](https://img.shields.io/badge/Deployment-Kubernetes-326CE5)](https://kubernetes.io)
[![Gateway: LiteLLM](https://img.shields.io/badge/Gateway-LiteLLM-green)](https://litellm.ai)
[![Frameworks: LangGraph · CrewAI · OpenClaw · Microsoft Agent Framework](https://img.shields.io/badge/Frameworks-LangGraph%20%C2%B7%20CrewAI%20%C2%B7%20OpenClaw%20%C2%B7%20Microsoft%20Agent%20Framework-orange)](https://github.com/intel/agent-toolkit)
<!-- vale on -->

# Intel® Agent Toolkit

**Quickly deploy a production-ready, open source agentic AI stack.**

> An agentic AI stack with an API gateway, inference routing, agent memory, sandboxed code execution, and observability. Toolkit Manager installs these pieces for you.

---

Teams that want **production AI agents** usually discover that a language-model endpoint is not enough. An agent has to remember a conversation, look up company data, call internal APIs, and sometimes run code it just generated. Doing that on a public cloud API also means prompts and documents leave the premises, and every team rebuilds the same plumbing: a gateway (LiteLLM), memory (Redis and pgvector), a sandbox, and traces (Langfuse).

Intel® Agent Toolkit is that plumbing on **Intel® Platforms** you already operate. [Toolkit Manager](https://github.com/intel/toolkit-manager) installs, updates, and tears it down, including the cluster and model serving. Applications keep an **OpenAI-compatible API**, so developers change the base URL and the API key and keep using LangGraph, CrewAI, OpenClaw, Microsoft Agent Framework, or a similar framework. Data and weights stay on your hardware. A GPU is not required, because memory, tools, and coordination run on the CPU, while a GPU or a cloud model can still answer the language-model call when you want that.

<p align="center">
  <img src="docs/assets/overview.png" alt="Intel® Agent Toolkit. Your agent (LangGraph, CrewAI, OpenClaw, Microsoft Agent Framework) calls an API gateway (LiteLLM, auth, rate limits). The stack provides memory (Redis, PostgreSQL, pgvector), inference and routing (vLLM, SGLang, cloud LLMs), an agent sandbox for code execution, and observability (Langfuse, Prometheus, Grafana). Deployment scales from one node to a rack with Ansible, Helm, and autoscaling, on a cluster (for example Kubernetes) or Docker Compose, using Intel® Xeon® processors for orchestration and execution, with GPUs and cloud LLMs for inference.">
</p>

---

## What you get

A default `install agent-toolkit` brings up platform, model serving, and this toolkit. Every component below is configurable: turn it off, change its settings, or bring your own. See [Configuration](docs/configuration/configuration.md) and [Meet the Agent Toolkit](docs/overview/meet.md#toolkit-components).

| Component | Default | Role |
|---|---|---|
| **LiteLLM** | On (`auth_type: litellm_vkey`) | HTTPS gateway: one URL, one master key, OpenAI-compatible `/v1/chat/completions`. Or Envoy gateway with Keycloak auth (`auth_type: jwt`) |
| **Redis Stack** | On | Short-term memory: session state, caches, and RediSearch |
| **PostgreSQL + pgvector** | On | Long-term memory: embeddings, RAG, and data that persists across restarts |
| **Agent Sandbox** | On | Isolated pods for agent actions, off the host |
| **OpenShell** | Off | Policy layer on the Agent Sandbox. Set `openshell_enabled: true` at install — [OpenShell](docs/reference/openshell.md) |
| **KubeRay** | Off | Distributes Python tasks across worker pods. Set `kuberay_enabled: true` |
| **Keycloak realm** | On | Creates an `agent-toolkit` realm on the platform Keycloak; does not deploy Keycloak itself. Set `keycloak_agent_config_enabled: false` to skip it |
| **Langfuse** | On with LiteLLM | Trace UI for each LLM call: prompt, tokens, latency, and the model that served the request |

Platform observability (Prometheus, Grafana, and related stack services) comes with the platform layer.

The toolkit is framework-agnostic. Agents built with LangGraph, CrewAI, Microsoft Agent Framework, OpenClaw, or any other agentic framework use it through standard interfaces: an OpenAI-compatible model gateway, Redis and PostgreSQL/pgvector for memory and state, and the Agent Sandbox. This repository provides the infrastructure those frameworks run on; it does not replace them.

### Validated frameworks

| Framework | Style |
|---|---|
| **LangGraph** | State-machine, multi-step agent workflows |
| **CrewAI** | Role-based multi-agent collaboration |
| **OpenClaw** | Local gateway for multi-agent routing via chat apps |
| **Microsoft Agent Framework** | Multi-agent and graph workflows in Python and .NET |

---

## Quickstart

Start here to run the agent platform on one host. Toolkit Manager installs the stack and one chat model. The last step calls the gateway so you can see that it answers. With the example domain `api.example.com`, the API is `https://litellm.api.example.com`.

Complete **[Prerequisites](docs/getting-started/prerequisites.md)** once before you start (hardware, kernel settings, SSH, DNS or `/etc/hosts`). Default catalog models need Intel® AMX (4th Gen Intel® Xeon® Scalable processors or newer); without it the model pod stays `Pending`. On older CPUs set `require_amx: false` in `env/local/models.yaml` after `init`.

### 1. Install Toolkit Manager and generate the configuration

<!-- golden-path: timeout 1800 -->
```bash
git clone https://github.com/intel/toolkit-manager.git
cd toolkit-manager
./toolkit_manager.sh configure
./toolkit_manager.sh init agent-toolkit
```

Run the rest of this quickstart from the `toolkit-manager` directory: `init` clones this repository under `ext/` and writes the configuration you edit next.

### 2. Set the configuration

`init` writes the files under `env/local/`. Set the domain there, and change the model only when you do not want the default, leaving the other keys as `init` wrote them. The default model needs about 131 GB of RAM; on a 128 GB server use `qwen2-5-coder-7b` or `llama-3-1-8b`.

| File | Setting | Purpose |
|---|---|---|
| `global_config.yaml` | `base_domain_name: api.example.com` | Domain. The API is `https://litellm.api.example.com`. |
| `config.agent-toolkit.yaml` | `auth_type: litellm_vkey` | LiteLLM gateway (default from `init`). Set `jwt` for Keycloak on the AI gateway. |
| `config.agent-toolkit.yaml` | `llm_services_autodeploy_models: ["qwen2-5-coder-14b"]` | Default chat model. Names must match `models.yaml`. |

`gateway_tls_mode: selfsigned` tells the installer to generate the certificate, and `storage_backend: local-path` keeps the data on this host.

### 3. Install

For a gated model (Llama, Gemma), export `HF_TOKEN` first. The default `qwen2-5-coder-14b` is not gated, so skip that.

<!-- golden-path: timeout 7200 -->
```bash
./toolkit_manager.sh install agent-toolkit
```

The first install usually takes 20–40 minutes, and the model pod can spend another 10–15 minutes downloading weights after that. If the command stops before it finishes, run it again and it will pick up at the step that did not complete. You use that same command later, whenever you change the YAML and want the cluster to match it.

### 4. Verify

<!-- golden-path: retry 60 30; timeout 2400; expect "choices" -->
```bash
export KUBECONFIG=$(pwd)/env/local/kubeconfig.yaml
kubectl get nodes
kubectl get pods -A

GW=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg-gateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
echo "$GW"
grep -q "litellm.api.example.com" /etc/hosts || \
  echo "$GW litellm.api.example.com langfuse.api.example.com" | sudo tee -a /etc/hosts

export LITELLM_MASTER_KEY=$(kubectl get secret litellm-master-key -n litellm \
  -o jsonpath='{.data.master_key}' | base64 -d)

# On a single node, curling the LoadBalancer IP from that same host often
# resets the TLS handshake (MetalLB hairpin). Port-forward for the smoke test;
# use https://litellm.api.example.com from any other machine.
kubectl -n litellm port-forward svc/litellm 4000:4000 >/tmp/litellm-pf.log 2>&1 &
PF=$!
trap 'kill $PF 2>/dev/null' EXIT
sleep 2

curl http://127.0.0.1:4000/v1/models \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}"

curl http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -d '{"model": "qwen2-5-coder-14b",
       "messages": [{"role": "user", "content": "Write a Python hello world"}],
       "max_tokens": 100}'
```

A working platform lists `qwen2-5-coder-14b` from `/v1/models` and returns the model's reply in `choices[0].message.content`. From another machine, call `https://litellm.api.example.com` with `-k` because the certificate was signed by the installer. Redis, pgvector, and the sandbox are already running behind that gateway, and their addresses inside the cluster are in the [backend connection reference](docs/reference/backend-connections.md).

| Symptom | Resolution |
|---|---|
| `Could not resolve host` | The `/etc/hosts` line is missing or uses the wrong IP. Rerun the `GW=` commands. |
| TLS `Connection reset` on this host | Expected on a single-node MetalLB setup. Use the port-forward curls above, or call the gateway from another machine. |
| `401` | `LITELLM_MASTER_KEY` is not set in this shell. Rerun the `export`. |
| `model not found` | Use the `id` returned by `/v1/models`. The default id is `qwen2-5-coder-14b`. |
| `500`, `503`, or a timeout | The model is still loading. Wait until its pod is `1/1 Running`, then retry. |
| Pods stuck, `too many open files` | The inotify sysctl from [Prerequisites](docs/getting-started/prerequisites.md#node-kernel-settings) was not applied on this node. |

Use the master key only in the curls above. For agents and apps, create a LiteLLM virtual key ([Access the model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md#access-the-model)) and call `https://litellm.api.example.com/v1`. Next: [build a first agent](docs/getting-started/first-agent.md). More detail: [single-node quickstart](docs/getting-started/quickstart.md) and [Troubleshooting](docs/faq-and-troubleshooting/troubleshooting.md).

---

## Choose how to run it

| Path | Best for | Guide |
|---|---|---|
| **Single node** | The agent platform on one machine. Start here. | [Single-node quickstart](docs/getting-started/quickstart.md) |
| **Multi-node** | One control plane plus workers | [Multi-node](docs/deployment/multi-node.md) |
| **Docker Compose** | Containers on one Linux host, without Toolkit Manager | [Docker](docs/deployment/docker.md) |

Also available: [rack-scale](docs/deployment/rack-scale.md) and [Intel® Arc™ GPU](docs/deployment/intel-arc.md) (both via Toolkit Manager).

---

## Documentation

The map of every guide is the [docs index](docs/README.md). Start here:

- [Meet the Agent Toolkit](docs/overview/meet.md)
- [Prerequisites](docs/getting-started/prerequisites.md)
- [Single-node quickstart](docs/getting-started/quickstart.md)
- [OpenShell](docs/reference/openshell.md) — policy layer on the Agent Sandbox (`openshell_enabled: true`)
- [Troubleshooting](docs/faq-and-troubleshooting/troubleshooting.md)

Related software Toolkit Manager can install: [Intel® Inference Toolkit](https://github.com/intel/inference-toolkit) and [Intel® Chatbot Toolkit](https://swcatalog.intel.com/developer/docs/enterprise-rag).

> Discover more AI software on the [Intel® Software Catalog](https://swcatalog.intel.com/).

---

## Contributing

Bug reports, doc fixes, and new components are all welcome. [CONTRIBUTING.md](CONTRIBUTING.md) starts with a table of what to do for each kind of change, then walks through your first pull request.

---

## License

Licensed under the [Apache License, Version 2.0](https://github.com/intel/agent-toolkit/blob/main/LICENSE).

---

*Intel, the Intel logo, Xeon, and Arc are trademarks of Intel Corporation or its subsidiaries.*
