# Prerequisites

[← Docs index](../README.md)

Do this once, on every server that will be part of the cluster, **before** the first `install`. When SSH and DNS (or `/etc/hosts`) are in place, go to the [single-node quickstart](quickstart.md). If you already know you will use several servers, still finish this page, then follow [multi-node deployment](../deployment/multi-node.md) instead of the single-node path.

## Hardware

These are the host minimums for a working install. A default install with `qwen2-5-coder-14b` requests about **131 GiB** of RAM. On a 128 GB node, choose `qwen2-5-coder-7b` or `llama-3-1-8b` in the [quickstart](quickstart.md#3-choose-a-model).

| Requirement | Minimum | Comfortable |
|---|---|---|
| **CPU** | 48 cores, x86_64 with Intel® AMX (4th Gen Intel® Xeon® Scalable processors or newer) | 96+ cores |
| **RAM** | 128 GB (with a smaller chat model) | 160 GB+ (default `qwen2-5-coder-14b`) |
| **Disk** | 150 GB free | 200 GB |
| **OS** | Ubuntu 22.04 or 24.04 | Ubuntu 22.04 LTS |
| **Processor** | Intel® Xeon® with AMX | Same family (validated) |

The CPU, RAM, and disk rows are the sizing. Catalog models set `require_amx: true` by default, so the model pod only schedules on AMX nodes; without AMX it stays `Pending` and the quickstart verify never answers. To opt out on older CPUs, set `require_amx: false` under `defaults:` in `env/local/models.yaml` after `init`. The OS can boot in far less RAM; the numbers above are for the stack plus a chat model.

> Tensor-parallel models need more memory in the model pod: about 128 GiB for `tp1`/`tp2`, 256 GiB for `tp >= 4`.

You also need:

- `sudo` on the host that runs the installer
- **No Docker on cluster nodes.** Cluster installs use containerd. Uninstall Docker before you run Toolkit Manager (it will not install while `docker.service` is present); Kubespray also deletes Docker and `/var/lib/docker` on those nodes. See [Toolkit Manager prerequisites](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/prerequisites.md). The [Docker Compose](../deployment/docker.md) path is separate and does not use Toolkit Manager.
- Network access to pull container images and Helm charts
- A [Hugging Face](https://huggingface.co) token with read access when a model is gated (Llama, Gemma). The default `qwen2-5-coder-14b` is not gated.

## Node kernel settings

`fs.inotify.max_user_instances` is a per-UID limit shared by every container running as that UID. The kernel default of 128 is exhausted on a node with many pods, and file watchers then fail with `too many open files` (the vLLM semantic router loses config reload). Raise it on every node before install:

```bash
sudo tee /etc/sysctl.d/90-inotify.conf > /dev/null <<'EOF'
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches   = 1048576
EOF
sudo sysctl --system
```

## 1. SSH keys

Log in as a **non-root** user with `sudo`. Do not run the installer as `root`. The installer talks to every cluster node over SSH, including a single-node install (it SSHs to itself).

On the host that runs the installer (single-node: this same machine):

<!-- golden-path: expect "SSH OK" -->
```bash
[ -f ~/.ssh/id_rsa ] || ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
ssh -o StrictHostKeyChecking=accept-new -o BatchMode=yes localhost true && echo "SSH OK"
```

Go on when that last command prints `SSH OK`. For several servers, put the same public key in `~/.ssh/authorized_keys` on every node and confirm passwordless login from the installer host:

```bash
ssh -i ~/.ssh/id_rsa <USERNAME>@<IP_ADDRESS>
```

If you use a bastion, the installer host must still reach the nodes through it.

After `init`, list machines in `env/<name>/nodes.yaml` (`nodes_control_plane`, `nodes_workers`). The installer regenerates `inventory/hosts.yaml` from that file on every run, so do not hand-edit `hosts.yaml` while `nodes.yaml` lists machines. For one server, leave the localhost defaults `init` wrote. For several servers, see [Install the platform](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/install-platform.md) and [Multi-node](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/multi-node.md) in Toolkit Manager (Agent Toolkit summary: [multi-node deployment](../deployment/multi-node.md#1-seed-config-then-list-every-node)).

## 2. DNS and TLS
<a id="dns-and-ssltls"></a>

The gateway is HTTPS. Browsers and `curl` need a name that matches the certificate.

`base_domain_name` is that name. Set it in Toolkit Manager `env/<name>/global_config.yaml` (default `inteltoolkits.ai`). With the default, these hostnames must resolve:

| Hostname | What it is |
|---|---|
| `litellm.inteltoolkits.ai` | LiteLLM UI and API (`auth_type: litellm_vkey`) |
| `langfuse.inteltoolkits.ai` | Langfuse tracing UI |
| `keycloak.inteltoolkits.ai` | Keycloak admin and UI sign-in (`auth_type: jwt`) |
| `grafana.inteltoolkits.ai` | Grafana dashboards |
| `ray.inteltoolkits.ai` | Ray Dashboard when KubeRay is on. [KubeRay](../reference/kuberay.md) |
| `vllm-sr.inteltoolkits.ai` | vLLM Semantic Router dashboard. Not in this release |

### Production

- Point DNS (or a load balancer) at the control-plane IP.
- Use a certificate from your CA that covers the base name **and** the subdomains (a wildcard `*.inteltoolkits.ai`, or a multi-SAN cert).
- Plan renewal before expiry. Port 80 may need to be open if your CA uses HTTP validation.

### Development (one server)

The default `gateway_tls_mode: selfsigned` creates the certificate during install.

**Install smoke test (curl).** You do not need `/etc/hosts` yet. The [quickstart verify](quickstart.md#5-confirm-it-works) calls the gateway with `--resolve` and `--cacert` once the LoadBalancer IP exists.

**Browsers and other machines.** After install, map the hostnames to the gateway IP (from `kubectl get svc` in the verify step). If you changed `base_domain_name`, use that domain here:

```bash
# Replace <GATEWAY_IP> with the LoadBalancer IP from the quickstart verify step.
sudo bash -c 'cat >> /etc/hosts <<EOF
<GATEWAY_IP> litellm.inteltoolkits.ai
<GATEWAY_IP> langfuse.inteltoolkits.ai
<GATEWAY_IP> keycloak.inteltoolkits.ai
<GATEWAY_IP> grafana.inteltoolkits.ai
<GATEWAY_IP> ray.inteltoolkits.ai
<GATEWAY_IP> vllm-sr.inteltoolkits.ai
EOF'
```

Optional: generate your own certificate instead of using `selfsigned`. OpenSSL 1.1.1 or later is required for `-addext`:

```bash
DOMAIN="inteltoolkits.ai"   # or your base_domain_name
mkdir -p ~/certs && cd ~/certs && \
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=${DOMAIN}" \
  -addext "subjectAltName = DNS:${DOMAIN}, DNS:litellm.${DOMAIN}, DNS:langfuse.${DOMAIN}, DNS:keycloak.${DOMAIN}, DNS:grafana.${DOMAIN}, DNS:ray.${DOMAIN}, DNS:vllm-sr.${DOMAIN}"
```

That writes `cert.pem` and `key.pem`. After `init`, set them in `env/local/global_config.yaml`:

```yaml
gateway_tls_mode: custom
gateway_tls_cert_file: /home/<user>/certs/cert.pem
gateway_tls_key_file: /home/<user>/certs/key.pem
```

If you add more hostnames later, regenerate the cert with extra `DNS:` entries.

## Next steps

When SSH is in place (and DNS or `/etc/hosts` for production), continue with the **[single-node quickstart](quickstart.md)** to install and send a chat request. Add `/etc/hosts` for browsers after the gateway IP exists.

If this host cannot reach the internet directly, set the proxy keys in [corporate proxy](../deployment/proxy.md) before you run `install`. For more than one server, use [multi-node deployment](../deployment/multi-node.md). [Meet the Agent Toolkit](../overview/meet.md) is the overview of what you are about to install; [Configuration](../configuration/configuration.md) lists every YAML field you can change later. Terms: [glossary](../reference/glossary.md).

---

*Intel, the Intel logo and Xeon are trademarks of Intel Corporation or its subsidiaries.*
