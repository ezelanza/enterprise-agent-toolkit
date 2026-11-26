# Intel® Agent Toolkit documentation

<p align="center">
  <img src="assets/banner.jpg" alt="Intel® Agent Toolkit">
</p>

The technical documentation for Intel® Agent Toolkit. Start with **Meet** for the what and why, **Get started** to stand it up, then **Deployment**, **Customize**, **Operate**, or **Reference** as you need them.

> [!IMPORTANT]
> Toolkit Manager installs and operates this toolkit. Clone **that** repository, run every command from its root, and `init agent-toolkit` will fetch this one into `ext/` for you. See [Meet the Agent Toolkit](overview/meet.md).

## Meet

| Guide | What it covers |
|---|---|
| [Meet the Agent Toolkit](overview/meet.md) | What it is, why it exists, when to choose it, and which repository to clone |
| [FAQ](faq-and-troubleshooting/frequently-asked-questions.md) | Short answers to the questions that come up most |
| [Glossary](reference/glossary.md) | Terms used across these guides, defined once |

## Get started

| Guide | What it covers |
|---|---|
| [Prerequisites](getting-started/prerequisites.md) | Hardware, SSH, DNS, and TLS to have ready before install |
| [Quickstart](getting-started/quickstart.md) | From `init agent-toolkit` to a working `/v1/chat/completions` call |
| [Build your first agent](getting-started/first-agent.md) | A LangChain agent with a tool, called through the gateway |

## Deployment

| Guide | What it covers |
|---|---|
| [Multi-node](deployment/multi-node.md) | One control plane plus worker nodes |
| [Rack-scale](deployment/rack-scale.md) | High-speed fabric via Toolkit Manager (`rack_scale_enabled`) |
| [Intel® Arc™ GPU](deployment/intel-arc.md) | GPU serving via Toolkit Manager (`vllm-xpu`) |
| [Docker Compose](deployment/docker.md) | Containers on one Linux host without Toolkit Manager |
| [Decommission](deployment/decommission.md) | Tear the install down and install again |

Behind a corporate proxy: [Corporate proxy](deployment/proxy.md). Related installs: [Inference Toolkit](https://github.com/intel/inference-toolkit) (`init inference`), [Chatbot Toolkit](https://github.com/intel/chatbot-toolkit) (`init erag`). Model catalog and `./model-manager deploy`: [Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md) in Toolkit Manager.

## Customize

Edit YAML under `env/local/` and re-run `install`.

| Guide | What it covers |
|---|---|
| [Configuration](configuration/configuration.md) | Which YAML file holds each setting |
| [Guardrails](configuration/guardrails.md) | Screen each prompt before the main model |
| [Routing](configuration/routing.md) · [Hybrid router](configuration/hybrid-router.md) | Send a request to one of several models (hybrid is on by default) |
| [vLLM semantic router](configuration/semantic-router.md) | Opt-in intent routing (`vllm_sr_enabled: true`) |

## Operate

| Guide | What it covers |
|---|---|
| [Validate the stack](faq-and-troubleshooting/validate.md) | Check the cluster after install |
| [Troubleshooting](faq-and-troubleshooting/troubleshooting.md) | Fix a failed install, API call, or pod |

## Reference

| Guide | What it covers |
|---|---|
| [Backend connections](reference/backend-connections.md) | Redis, pgvector, sandbox, and gateway URLs |
| [Agent sandbox](reference/agent-sandbox.md) | Isolated pods for agent-generated code |
| [OpenShell](reference/openshell.md) | Policy around that code (off by default) |
| [KubeRay](reference/kuberay.md) | Ray jobs in the cluster (off by default) |

To report a bug or change the toolkit, see [Contributing](../CONTRIBUTING.md).

---

*Intel, the Intel logo, and Arc are trademarks of Intel Corporation or its subsidiaries.*
