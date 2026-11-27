# Rack-scale AI fabric

[← Docs index](../README.md)

Deploy the Agent Toolkit on a rack through Toolkit Manager. The rack is described in `env/<name>/rack-profile.yaml`, and one `install agent-toolkit` sets up the fabric, Kubernetes, Ceph, and the toolkit. Clos, L2, and `none` (no fabric) racks are validated in this release; star is not.

## What to use

Use one env for every command (`--env rack` here):

1. `./toolkit_manager.sh init agent-toolkit --env rack`
2. In `env/rack/global_config.yaml`, set `rack_scale_enabled: true`, `metallb_ip_range`, and `storage_backend: ceph`. Write `env/rack/rack-profile.yaml` and leave `env/rack/nodes.yaml` empty. [Rack-scale deployment](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/rack-scale.md) in Toolkit Manager covers these settings, the topologies, GPU nodes, validation, and troubleshooting.
3. `./toolkit_manager.sh install agent-toolkit --env rack`

To add or remove a worker later, edit `rack-profile.yaml`, not `nodes.yaml`. See [Node scaling](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/node-scaling.md#rack-scale) in Toolkit Manager.

For a multi-host install without the fabric profile, use [Multi-node deployment](multi-node.md).

## Next steps

- [Rack-scale deployment](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/rack-scale.md) — Toolkit Manager
- [Multi-node deployment](multi-node.md) — Ethernet multi-node without the fabric profile
- [Prerequisites](../getting-started/prerequisites.md) — host prep before install

---

*Intel, the Intel logo and Xeon are trademarks of Intel Corporation or its subsidiaries.*
