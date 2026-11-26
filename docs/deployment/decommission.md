# Decommission and redeploy

[← Docs index](../README.md)

Use this page when you want to remove the Agent Toolkit from a cluster and install it again, for example after a failed run, or when you want a different set of models. Teardown deletes the workloads, persistent volumes, and secrets for the target you name. That cannot be undone. The YAML under `env/<env>/` is **kept**, so your domain, TLS paths, and model flags survive and you do not have to re-type them.

## 1. Tear down

From the Toolkit Manager directory (see [which repository to start from](../overview/meet.md#which-repository-to-start-from)):

```bash
./toolkit_manager.sh teardown agent-toolkit
```

That removes the Agent Toolkit layer only — the components this repo owns (`redis`, `pgvector`, `keycloak_agent_config`, agent sandbox / OpenShell, KubeRay when on) and anything that depends on that layer — and deletes objects labelled `toolkit-manager.intel.com/env` and `toolkit-manager.intel.com/component`. Cluster, platform, and **inference stay** (LiteLLM and Langfuse are inference-layer; they are not removed). See Toolkit Manager [Agent Toolkit](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/agent-toolkit.md) (“keeps cluster, platform and inference intact”) and [teardown](https://github.com/intel/toolkit-manager/blob/main/docs/reference/cli.md#teardown).

To tear down **everything** the installer created, including Kubernetes itself:

```bash
./toolkit_manager.sh teardown infrastructure
```

`teardown infrastructure` removes the cluster. Use it only when you are ready to lose Kubernetes, not just the toolkit. The YAML under `env/<name>/` is kept.

## 2. Install again

If you want a different layout, edit `env/local/*.yaml` first (domain, models, storage, and so on). Then:

```bash
./toolkit_manager.sh install agent-toolkit
```

`install` is idempotent. After it finishes:

```bash
./toolkit_manager.sh validate agent-toolkit
```

## Next steps

To stand the stack up again from scratch on one server, follow the [single-node quickstart](../getting-started/quickstart.md). The fields you can change before the next `install` are listed in [Configuration](../configuration/configuration.md). For several servers, use [multi-node deployment](multi-node.md). If install fails after redeploy, see [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md).
