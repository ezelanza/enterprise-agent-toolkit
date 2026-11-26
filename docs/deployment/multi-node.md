# Multi-node deployment

[← Docs index](../README.md)

Use this guide when you have more than one host and you want the Agent Toolkit spread across them. A typical layout is one **control-plane** node that runs Kubernetes and the stack services, plus one or more **worker** nodes that take extra model pods when a single host is not enough CPU or memory.

Kubernetes itself is installed by Kubespray. List machines in `env/<name>/nodes.yaml` (`nodes_control_plane`, `nodes_workers`); the installer regenerates `inventory/hosts.yaml` from that file on every run.

If you only have one host, follow the [single-node quickstart](../getting-started/quickstart.md) instead.

For a high-speed fabric and NVMe Ceph, see [Rack-scale deployment](rack-scale.md) instead of this Ethernet multi-node path.

Complete **[Prerequisites](../getting-started/prerequisites.md)** on every node before you start: SSH from the installer host to all nodes, `/etc/hosts` (or DNS), and TLS.

## Topologies

Most teams start with a single control plane and add workers for inference. Three control-plane nodes are only needed if you cannot tolerate losing the Kubernetes API on one node.

| Layout | When to choose it |
|---|---|
| 1 control plane + N workers | Production that can scale models out. The control plane also runs as a worker unless you taint it. |
| 3 control planes + N workers | High availability for etcd and the API server. Put a load balancer in front and point `base_domain_name` at the load-balancer IP. |

The gateway (NGINX ingress or Envoy) binds to the control-plane node by default. With several control planes, traffic should land on the load balancer, not on a single node's IP.

## 1. Seed config, then list every node

Run these from the Toolkit Manager directory, on the host that will drive the install. [Meet the Agent Toolkit](../overview/meet.md) explains that Toolkit Manager installs, updates, and tears the toolkit down.

```bash
chmod +x toolkit_manager.sh
./toolkit_manager.sh configure
./toolkit_manager.sh init agent-toolkit
```

`init` seeds `env/local/nodes.yaml` for a single localhost node. For a real cluster, edit that file. The installer writes `inventory/hosts.yaml` from it on every run, so do not hand-edit `hosts.yaml` while `nodes.yaml` lists machines.

Here is a control plane plus two workers. Replace IPs, hostnames, the SSH user, and the key path with your values:

```yaml
# env/local/nodes.yaml
nodes_control_plane:
  - ip: 10.0.0.1
    hostname: master1

nodes_workers:
  - ip: 10.0.0.2
    hostname: worker1
  - ip: 10.0.0.3
    hostname: worker2

nodes_ssh_user: "ubuntu"
nodes_ssh_key: "/home/ubuntu/.ssh/id_rsa"   # chmod 600
```

The host that runs `install` must be able to SSH to every node with that key. Password logins are not enough; Ansible will fail. Jump hosts and hand-written Kubespray inventory are covered in Toolkit Manager: [Install the platform](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/install-platform.md) and [Multi-node](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/multi-node.md).

Open these ports between nodes: Kubernetes API `6443`, etcd `2379–2380`, Calico BGP `179`, kubelet `10250`. Keep the NodePort range `30000–32767` off the public internet. Redis in this stack is ClusterIP on purpose; do not publish it as NodePort.

## 2. DNS and TLS on every node

Each node needs to resolve the gateway names. If you do not have DNS yet, add `/etc/hosts` entries that point at the control-plane IP (or the load-balancer IP):

```bash
# 10.0.0.1 = control-plane or LB. Use your base_domain_name (default inteltoolkits.ai).
sudo bash -c 'cat >> /etc/hosts <<EOF
10.0.0.1 litellm.inteltoolkits.ai
10.0.0.1 langfuse.inteltoolkits.ai
10.0.0.1 keycloak.inteltoolkits.ai
10.0.0.1 grafana.inteltoolkits.ai
EOF'
```

On the **control-plane**, create a certificate that covers those names (or install one from your CA), then point `global_config.yaml` at the files:

```bash
DOMAIN="inteltoolkits.ai"   # or your base_domain_name
mkdir -p ~/certs && cd ~/certs && \
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=${DOMAIN}" \
  -addext "subjectAltName = DNS:${DOMAIN}, DNS:litellm.${DOMAIN}, DNS:langfuse.${DOMAIN}, DNS:keycloak.${DOMAIN}, DNS:grafana.${DOMAIN}, DNS:*.${DOMAIN}"
```

```yaml
# env/local/global_config.yaml
gateway_tls_mode: custom
gateway_tls_cert_file: /home/ubuntu/certs/cert.pem
gateway_tls_key_file: /home/ubuntu/certs/key.pem
```

The same `base_domain_name` must match DNS/`/etc/hosts` and the certificate. Details and the production CA path are in [Prerequisites](../getting-started/prerequisites.md#2-dns-and-tls).

## 3. The rest of the YAML

The files under `env/local/` are the same ones the [quickstart](../getting-started/quickstart.md) uses. On several nodes, the settings that usually change are the domain, TLS (already set above), and **storage**. `local-path` only works on one node; use `nfs` or `ceph` when pods can land on any worker.

```yaml
# env/local/global_config.yaml
base_domain_name: inteltoolkits.ai
storage_backend: ceph
```

```yaml
# env/local/config.agent-toolkit.yaml
auth_type: litellm_vkey
llm_services_autodeploy_models: ["qwen2-5-coder-14b"]
```

The catalog names are in the [quickstart](../getting-started/quickstart.md#3-choose-a-model).

If a model is gated on Hugging Face (Llama, Gemma), export a token in the same shell as `install`:

```bash
export HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx
```

`qwen2-5-coder-14b` is ungated and does not need a token. It does need about 96 GiB for the model pod.

## 4. Install from the control plane

```bash
./toolkit_manager.sh install agent-toolkit
```

Kubespray reads the generated `hosts.yaml` and provisions Kubernetes first (often 20–30 minutes on a fresh set of nodes). After that, the installer deploys platform, inference, and the toolkit. The command is idempotent: if it stops part-way, run it again.

## 5. Confirm the cluster and the API

Wait until every node is Ready. The vLLM pod may stay in `ContainerCreating` for another 10–15 minutes while it downloads weights.

```bash
export KUBECONFIG=$(pwd)/env/local/kubeconfig.yaml
kubectl get nodes -o wide
kubectl get pods -A

GATEWAY_IP=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg-gateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
DOMAIN=$(yq '.base_domain_name' env/local/global_config.yaml)

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

Same verify as the README and single-node quickstart. Redis, pgvector, and the sandbox are already running. In-cluster URLs are in the [quickstart](../getting-started/quickstart.md#what-also-came-up).

## Add a worker later

Do **not** re-run `install` to grow a running cluster. Use Toolkit Manager node scaling:

1. Add the worker under `nodes_workers` in `env/<name>/nodes.yaml`.
2. Set up passwordless SSH from the installer host to the new node, the same way as [Prerequisites](../getting-started/prerequisites.md#1-ssh-keys).
3. Preview, then apply:

   ```bash
   ./toolkit_manager.sh add-node --dry-run
   ./toolkit_manager.sh add-node
   ```

To remove a worker, delete it from `nodes_workers`, then run `./toolkit_manager.sh remove-node --dry-run` and `./toolkit_manager.sh remove-node`. Full rules: [Node scaling](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/node-scaling.md) and [CLI](https://github.com/intel/toolkit-manager/blob/main/docs/reference/cli.md) in Toolkit Manager.

## Notes

- **Redis** runs as a single StatefulSet in this release, shared by all agents. It is not a Redis Cluster. High availability is a chart-values change, not the default.
- **Model weights** live on the shared `model-store` PVC (`pvc_name` default `model-store`, size `llm_services_model_store_size`, default `200Gi`). On multi-node that volume must be ReadWriteMany (`storage_backend: nfs`, `ceph`, `netapp-trident`, or a custom RWX class — not `local-path`). See Toolkit Manager [Storage](https://github.com/intel/toolkit-manager/blob/main/docs/configuration/configuration.md#storage) and Inference Toolkit [Model catalog](https://github.com/intel/inference-toolkit/blob/main/docs/configuration/catalog.md#model-store-size).
- **Firewall:** in addition to the ports above, do not expose `30000–32767` to the internet. Leave Redis as ClusterIP.

## Troubleshooting

| Symptom | What to try |
|---|---|
| Node `NotReady` | On that node, run `systemctl status kubelet` and check the kubelet logs. |
| Kubespray SSH refused | Confirm IPs and `nodes_ssh_key` (or per-node `ssh_key`) in `nodes.yaml`, and that you can `ssh` by hand with that key. |
| Pods stuck `Pending` | `kubectl describe node <node>`: usually CPU, memory, or a taint that blocks scheduling. |
| Model pod never lands on a worker | Same check on that worker: resources and taints. |

## Next steps

Once nodes are Ready and chat completions work, you can change YAML in [Configuration](../configuration/configuration.md), copy in-cluster URLs from the [backend connection reference](../reference/backend-connections.md), or [tear the stack down](decommission.md) and install again. The first-run path on one host is the [single-node quickstart](../getting-started/quickstart.md). Failures: [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md).
