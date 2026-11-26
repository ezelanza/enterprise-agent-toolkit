# Rack-scale AI fabric

[← Docs index](../README.md)

Deploy the Agent Toolkit on a rack with a dedicated high-speed fabric through Toolkit Manager.

## What to use

1. Enable rack-scale in Toolkit Manager (`rack_scale_enabled`, `env/<name>/rack-profile.yaml`).
2. Follow [Rack-scale deployment](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/rack-scale.md) in Toolkit Manager for fabric topologies, Ceph on the fabric, GPU nodes, validation, and troubleshooting.
3. Then install the Agent Toolkit layer as usual (`init agent-toolkit` / `install agent-toolkit`).

For a multi-host install without the fabric profile, use [Multi-node deployment](multi-node.md).

## Next steps

- [Rack-scale deployment](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/rack-scale.md) — Toolkit Manager
- [Multi-node deployment](multi-node.md) — Ethernet multi-node without the fabric profile
- [Prerequisites](../getting-started/prerequisites.md) — host prep before install

---

*Intel, the Intel logo and Xeon are trademarks of Intel Corporation or its subsidiaries.*
