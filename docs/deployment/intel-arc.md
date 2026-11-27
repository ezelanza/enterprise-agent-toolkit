# Intel® Arc™ GPU

[← Docs index](../README.md)

Serve models on Intel® Arc™ GPUs through Toolkit Manager, including Intel® Arc™ Pro B-Series (B70) and other Xe GPUs.

## What to use

1. Install the stack with Toolkit Manager as usual ([Prerequisites](../getting-started/prerequisites.md), then the [quickstart](../getting-started/quickstart.md) or [multi-node](multi-node.md)). On GPU hosts, Toolkit Manager installs the Intel GPU device plugin when Node Feature Discovery sees an Intel GPU — details for Arc Pro B-Series / B70 and other Xe GPUs: [Install the platform](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/install-platform.md) in Toolkit Manager.
2. Deploy a catalog model that uses the `vllm-xpu` runtime, for example `vllm-xpu-llama-3-1-8b` or `vllm-xpu-gpt-oss-20b`:

   ```bash
   ./model-manager deploy vllm-xpu-llama-3-1-8b
   # or: ./model-manager deploy <name> --server vllm-xpu
   ```

   Catalog entries, flags, and access steps: [Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md) in Toolkit Manager.

3. For the `vllm-xpu` engine itself (images, shared settings, which models may use it): [Runtimes](https://github.com/intel/inference-toolkit/blob/main/docs/configuration/runtimes.md) in Intel® Inference Toolkit.

## Next steps

- [Deploy a model](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/deploy-models.md) — `model-manager` and the catalog
- [Runtimes](https://github.com/intel/inference-toolkit/blob/main/docs/configuration/runtimes.md) — `vllm-xpu` and other engines
- [Configuration](../configuration/configuration.md) — Agent Toolkit YAML under `env/local/`

---

*Intel, the Intel logo, and Arc are trademarks of Intel Corporation or its subsidiaries.*
