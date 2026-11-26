# Agent sandbox

[← Docs index](../README.md)

Agent-generated code must not run on the host or inside the model-serving pod. **Agent sandbox** runs it in a short-lived Kubernetes pod with its own filesystem, process tree, and network. The agent calls a router service, and the router creates the pod. Use this page to configure the sandbox, check it, add runtimes, and call it from the Python SDK. The implementation is [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox).

The sandbox is installed with `install agent-toolkit` and is on by default. For the in-cluster URL only, see the [backend connection reference](backend-connections.md#agent-sandbox).

## How it works

```
Agent (e.g. Coding Agent)
        │  HTTP (in-cluster DNS)
        ▼
sandbox-router-svc.agent-sandbox-system.svc.cluster.local:8080
        │  creates a SandboxClaim through the Kubernetes API
        ▼
agent-sandbox-controller
        │  starts a pod from the SandboxTemplate
        ▼
sandbox-claim-<id>   (NetworkPolicy: ingress only from app=sandbox-router, same namespace)
        │
        ▼
python-runtime-sandbox: uvicorn on :8888
        POST /execute {"command": "..."} → {stdout, stderr, exit_code}
```

- Every component runs in the **`agent-sandbox-system`** namespace. The controller generates a NetworkPolicy that admits traffic only from `app=sandbox-router` pods in the same namespace, so templates and sandboxes must live there too.
- The router is a Go reverse proxy (`sandbox-router-go`). This install turns on `--authz-mode=tokenreview`, so each request to the router must carry a Kubernetes service-account Bearer token. There is no static `ROUTER_AUTH_TOKEN` secret. See [Router token](#router-token).
- Sandbox create and delete go through the Kubernetes API (CRDs). The router only forwards traffic to a running sandbox pod.
- Sandbox pods get an egress NetworkPolicy (`eat-sandbox-pool-egress`): they may reach the public internet, but **not** RFC1918 or link-local ranges. Code inside a sandbox cannot call in-cluster Redis, pgvector, or LiteLLM by ClusterIP. Reach those services from the agent pod, not from sandbox code.
- A default install also creates a warm pool of **5** idle Python sandbox pods (`agent_sandbox_python_warm_pool_size`). That uses CPU and memory before the first tool call. Set the size to `0` to skip pre-warming.
- All images come from published registries. No local build is required.

| Component | Kind | Namespace | Image or detail |
|---|---|---|---|
| `agent-sandbox-controller` | Deployment | `agent-sandbox-system` | `registry.k8s.io/agent-sandbox/agent-sandbox-controller:v1.0.10` |
| CRDs | Cluster-scoped | | `sandboxes`, `sandboxtemplates`, `sandboxclaims`, `sandboxwarmpools` (v1beta1) |
| `sandbox-router` | Deployment | `agent-sandbox-system` | `registry.k8s.io/agent-sandbox/sandbox-router-go:v1.0.10` (TokenReview auth) |
| `sandbox-router-svc` | Service | `agent-sandbox-system` | ClusterIP :8080 |
| `python-sandbox-template` | SandboxTemplate | `agent-sandbox-system` | `registry.k8s.io/agent-sandbox/python-runtime-sandbox:v1.0.10` |
| `python-pool` | SandboxWarmPool | `agent-sandbox-system` | 5 pre-warmed pods from `python-sandbox-template` |

## 1. Configure

After `./toolkit_manager.sh init agent-toolkit`, set these in `env/local/config.agent-toolkit.yaml`. The defaults need no change.

| Field | Default | What it means |
|---|---|---|
| `agent_sandbox_enabled` | `true` | Set `false` to skip the sandbox. |
| `agent_sandbox_namespace` | `agent-sandbox-system` | Namespace for every sandbox component. |
| `agent_sandbox_version` | `v1.0.10` | Upstream release. Selects the manifests fetched from GitHub and the runtime image tag. |
| `agent_sandbox_router_image` | `registry.k8s.io/agent-sandbox/sandbox-router-go:<version>` | Router image. Override to mirror it for air-gapped clusters. |
| `agent_sandbox_python_runtime_image` | `registry.k8s.io/agent-sandbox/python-runtime-sandbox:<version>` | Image for the default template. Override to mirror it. |
| `agent_sandbox_python_warm_pool_name` | `python-pool` | Name of the default warm pool. |
| `agent_sandbox_python_warm_pool_size` | `5` | Pre-warmed pods kept ready for the first call. Uses node CPU and memory on install. `0` creates no warm pool. |

Cluster-wide settings that also apply (`base_domain_name`, `gateway_tls_mode`) are in `env/local/global_config.yaml`. API sign-in is `auth_type` in `config.agent-toolkit.yaml`. See [Configuration](../configuration/configuration.md).

<a id="router-token"></a>

## Router token

Every data-plane call through the router (command execution after `create_sandbox`) needs a Kubernetes service-account Bearer token. `/healthz` does not. Mint a short-lived token from a service account that can talk to the API (the default SA in the sandbox namespace is enough for a lab):

```bash
export ROUTER_TOKEN=$(kubectl create token default -n agent-sandbox-system --duration=1h)
```

Pass it on every SDK client that talks to the router with `extra_headers` (Direct mode). Tunnel mode in SDK 1.0.10 has no `extra_headers` field, so from a workstation use a port-forward plus Direct mode:

```bash
kubectl port-forward -n agent-sandbox-system svc/sandbox-router-svc 8080:8080
```

```python
from k8s_agent_sandbox import SandboxClient
from k8s_agent_sandbox.models import SandboxDirectConnectionConfig

client = SandboxClient(
    connection_config=SandboxDirectConnectionConfig(
        api_url="http://localhost:8080",
        extra_headers={"Authorization": f"Bearer {ROUTER_TOKEN}"},
    )
)
```

In an agent pod, use the same `extra_headers` with `api_url="http://sandbox-router-svc.agent-sandbox-system.svc.cluster.local:8080"`, and prefer the pod's projected service-account token (`/var/run/secrets/kubernetes.io/serviceaccount/token`) instead of `kubectl create token`.

## 2. Install

```bash
./toolkit_manager.sh configure              # one-time
./toolkit_manager.sh init agent-toolkit     # writes env/local/
./toolkit_manager.sh install agent-toolkit  # installs the sandbox with the layer
```

The installer:

1. Creates the namespace and applies the upstream `sandbox-with-extensions` manifest (CRDs and controller).
2. Waits for the controller, then applies the published `sandbox-router` manifests and pins the router image. It also sets TokenReview auth (`--authz-mode=tokenreview`).
3. Applies `python-sandbox-template` and creates the `python-pool` warm pool.

Re-run `install` to resume a failed run or to re-apply the same manifests. The playbook does not skip the sandbox when the controller already exists.

> The default `--env` is `local`; pass a different `--env <name>` to target
> another environment. `teardown` and `validate <target>` verbs are also
> available.

## 3. Verify

```bash
kubectl get pods -n agent-sandbox-system                                    # all Running
kubectl get deployment -n agent-sandbox-system agent-sandbox-controller sandbox-router

kubectl get crd sandboxes.agents.x-k8s.io -o jsonpath='{.spec.versions[*].name}'
# Expected: v1alpha1 v1beta1

kubectl get sandboxtemplate -n agent-sandbox-system                         # python-sandbox-template
kubectl get sandboxwarmpool -n agent-sandbox-system python-pool             # 5 ready

kubectl port-forward -n agent-sandbox-system svc/sandbox-router-svc 8080:8080 &
curl -s http://localhost:8080/healthz
# {"status":"ok"}
```

## 4. Run code from the SDK

Install the SDK version that matches the deployment. This release pins agent-sandbox **v1.0.10**, so use the matching SDK:

```bash
pip install k8s-agent-sandbox==1.0.10
```

SDK documentation: [agentic-sandbox-client](https://github.com/kubernetes-sigs/agent-sandbox/tree/v1.0.10/clients/python/agentic-sandbox-client). Mint a [Router token](#router-token) first; every example below needs it on `extra_headers`.

| Where the client runs | Connection |
|---|---|
| Agent pod in the cluster | Direct mode to `sandbox-router-svc…:8080` plus a Bearer token |
| Workstation | `kubectl port-forward` to the router Service, then Direct mode to `http://localhost:8080` plus a Bearer token |

**In the cluster (agent pods):**

```python
from pathlib import Path
from k8s_agent_sandbox import SandboxClient
from k8s_agent_sandbox.models import SandboxDirectConnectionConfig

token = Path("/var/run/secrets/kubernetes.io/serviceaccount/token").read_text().strip()

client = SandboxClient(
    connection_config=SandboxDirectConnectionConfig(
        api_url="http://sandbox-router-svc.agent-sandbox-system.svc.cluster.local:8080",
        extra_headers={"Authorization": f"Bearer {token}"},
    )
)

sandbox = client.create_sandbox(
    template="python-sandbox-template",
    namespace="agent-sandbox-system",
)
try:
    result = sandbox.commands.run("python3 -c \"print('Hello from in-cluster sandbox!')\"")
    print(result.stdout)
finally:
    sandbox.terminate()
```

**From a workstation.** Keep a port-forward running, mint `ROUTER_TOKEN` as in [Router token](#router-token), then:

```python
import os
from k8s_agent_sandbox import SandboxClient
from k8s_agent_sandbox.models import SandboxDirectConnectionConfig

client = SandboxClient(
    connection_config=SandboxDirectConnectionConfig(
        api_url="http://localhost:8080",
        extra_headers={"Authorization": f"Bearer {os.environ['ROUTER_TOKEN']}"},
    )
)

sandbox = client.create_sandbox(
    template="python-sandbox-template",
    namespace="agent-sandbox-system",
)
try:
    result = sandbox.commands.run("python3 --version")
    print(result.stdout)
finally:
    sandbox.terminate()
```

**Session sandbox (state across calls).** To keep installed packages and files between tool calls, create the sandbox once and reuse it (same `client` with the token):

```python
_sandbox = None

def get_sandbox():
    global _sandbox
    if _sandbox is None or not _sandbox.is_active:
        _sandbox = client.create_sandbox(
            template="python-sandbox-template",
            namespace="agent-sandbox-system",
        )
    return _sandbox

get_sandbox().commands.run("pip install pandas")
result = get_sandbox().commands.run("python3 -c 'import pandas; print(pandas.__version__)'")
print(result.stdout)

get_sandbox().terminate()
_sandbox = None
```

## 5. Add a runtime template

Add one `SandboxTemplate` per runtime (Node.js, R, Java, and so on).

1. Build and push an image that serves the same API as `python-runtime-sandbox` on port 8888: `POST /execute` with `{"command": "<shell command>"}`, returning `{"stdout": "...", "stderr": "...", "exit_code": 0}`.

   ```bash
   docker build -t your-registry.io/my-node-sandbox:v1.0 ./path/to/dockerfile-dir
   docker push your-registry.io/my-node-sandbox:v1.0
   ```

   For local development without a registry, load the image instead: `kind load docker-image <image>` (Kind) or `minikube image load <image>` (Minikube).

2. Create the template in `agent-sandbox-system`, the namespace where sandboxes are created. This is the default template with a different name and image:

   ```yaml
   apiVersion: extensions.agents.x-k8s.io/v1beta1
   kind: SandboxTemplate
   metadata:
     name: node-sandbox-template
     namespace: agent-sandbox-system
   spec:
     service: true          # headless Service per pod: required for SDK DNS mode
     podTemplate:
       spec:
         containers:
         - name: runtime
           image: your-registry.io/my-node-sandbox:v1.0
           imagePullPolicy: IfNotPresent   # or Always for frequent updates
           ports:
           - name: api
             containerPort: 8888
             protocol: TCP
           resources:
             requests: { cpu: "250m", memory: "256Mi" }
             limits:   { cpu: "1000m", memory: "1Gi" }
           securityContext:
             allowPrivilegeEscalation: false
             runAsNonRoot: true
             runAsUser: 1000
             capabilities:
               drop: [ALL]
   ```

3. Apply it and use it:

   ```bash
   kubectl apply -f node-sandbox-template.yaml
   kubectl get sandboxtemplate -n agent-sandbox-system
   ```

   ```python
   sandbox = client.create_sandbox(template="node-sandbox-template", namespace="agent-sandbox-system")
   ```

## 6. Add a warm pool

A warm pool keeps idle sandbox pods ready, so `create_sandbox` returns without waiting for scheduling or an image pull. The controller hands out a warm pod and refills the pool in the background. The installer creates `python-pool`; to size it, use `agent_sandbox_python_warm_pool_size` before install. For another template, apply a pool in the same namespace as the template:

```yaml
apiVersion: extensions.agents.x-k8s.io/v1beta1
kind: SandboxWarmPool
metadata:
  name: node-pool
  namespace: agent-sandbox-system
  labels:
    app: node-pool
spec:
  replicas: 3                          # pre-warmed pods to keep
  sandboxTemplateRef:
    name: node-sandbox-template        # must exist in the same namespace
```

```bash
kubectl apply -f warmpool.yaml
kubectl get sandboxwarmpool -n agent-sandbox-system
```

## Kata Containers

A default sandbox pod uses the `runc` runtime. It is isolated by namespaces and cgroups and shares the host kernel with every other workload on the node. [Kata Containers](https://github.com/kata-containers/kata-containers) boots each pod in its own virtual machine and guest kernel.

| RuntimeClass | Hypervisor | Notes |
|---|---|---|
| `kata-fc` | [Firecracker](https://github.com/kata-containers/kata-containers/tree/main/docs/design/arch-images#firecracker) | Small device model, about 125 ms boot. Needs the containerd devmapper snapshotter |
| `kata-qemu` | QEMU | Broader devices (VFIO and device passthrough). Heavier boot. Uses the default snapshotter |

The template follows the upstream [Firecracker sandbox example](https://agent-sandbox.sigs.k8s.io/docs/use-cases/examples/firecracker-sandbox/) and keeps the published `python-runtime-sandbox` image, so the `k8s-agent-sandbox` SDK is unchanged. The Kata template differs by `runtimeClassName`.

### Enable Kata

```yaml
# env/local/config.agent-toolkit.yaml
agent_sandbox_enabled: true

# env/local/global_config.yaml
kata_containers_enabled: true
kata_runtime: "kata-fc"       # kata-fc | kata-qemu | both
kata_version: "3.32.0"        # kata-deploy Helm chart version
```

Kata is an add-on. The `SandboxTemplate` it publishes needs the agent-sandbox CRDs. With `kata_containers_enabled: true` and `agent_sandbox_enabled: false`, the runtime classes are still installed and no Kata template or pool is created.

### Node prerequisites

| Requirement | Needed by | What the playbook does |
|---|---|---|
| `/dev/kvm` (Intel VT-x on bare metal, or nested virtualization on a VM) | both runtimes | `stat`. If the device is missing, the playbook loads `kvm` and `kvm_intel` when the CPU has the `vmx` flag, then checks again. If `vmx` is absent, it warns and skips the host |
| containerd devmapper snapshotter in `ok` state | `kata-fc` only | Provisioned by the playbook. It creates a loopback thin pool at `/var/lib/containerd/devmapper` (`containerd-devmapper.service`, so it survives reboot), registers the snapshotter in `/etc/containerd/config.toml`, and restarts containerd on each KVM-capable node |

Firecracker cannot use overlayfs for the container rootfs. It needs devmapper. Kubespray rewrites `/etc/containerd/config.toml` on a fresh install, and the Kata step runs after that, so it writes the snapshotter again on every deploy. The thin pool defaults to a 100 GB sparse file. To use a real block device, pre-create a `containerd-thinpool` device and the playbook reuses it.

| Situation | Result |
|---|---|
| No node exposes `/dev/kvm` | Stops. Enable VT-x, or set `kata_containers_enabled: false` |
| `kata_runtime` is `kata-fc` or `both`, and devmapper is not `ok` on any node | Stops. Fix the thin pool, or set `kata_runtime: kata-qemu` |
| Firecracker is on and devmapper is not `ok` on some nodes | Those nodes are not labelled. A label from an earlier run is removed. The run names them |
| `agent_sandbox_enabled: false` | Runtime classes are installed. No Kata template or pool |
| `kata_runtime` is not one of the three values | Stops and lists the valid values |

### What gets installed

| Resource | Kind | Namespace |
|---|---|---|
| `containerd-thinpool` and `containerd-devmapper.service` | devmapper thin pool (`kata-fc` only, per KVM node) | host |
| `kata-deploy` | DaemonSet (upstream Helm chart) | `kube-system` |
| `kata-fc` / `kata-qemu` | RuntimeClass | cluster |
| `ei-kata-eligible=true` | Node label, on nodes that passed the preflight | — |
| `kata-sandbox-template` | SandboxTemplate | `agent-sandbox-system` |
| `kata-pool` (size 1) | SandboxWarmPool | `agent-sandbox-system` |

The chart is `oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy`. `kata-deploy` installs the Kata binaries, guest kernel, and rootfs on each node and restarts containerd there. The rollout wait is 15 minutes.

The warm pool is created without a readiness gate, so a virtualization problem on one node does not hang the stack install. Check it with the commands below.

### Verify Kata

```bash
kubectl get ds -n kube-system kata-deploy
kubectl get runtimeclass                                  # kata-fc and/or kata-qemu
kubectl get nodes -l ei-kata-eligible=true
kubectl get sandboxtemplate,sandboxwarmpool -n agent-sandbox-system

kubectl get pod -n agent-sandbox-system -l sandbox=kata \
  -o jsonpath='{.items[0].spec.runtimeClassName}{"\n"}'   # kata-fc
```

From the SDK, use the same client as in [Run code from the SDK](#4-run-code-from-the-sdk) (with the [Router token](#router-token) on `extra_headers`). Only the warm pool name changes:

```python
sandbox = client.create_sandbox(
    warmpool="kata-pool",
    namespace="agent-sandbox-system",
)
try:
    # A microVM has its own guest kernel. This differs from `uname -r` on the host.
    print(sandbox.commands.run("uname -r").stdout)
finally:
    sandbox.terminate()
```

| Symptom | What to do |
|---|---|
| `FailedCreatePodSandBox ... ENOENT` on a `kata-fc` pod | Devmapper is not active on that node. `systemctl status containerd-devmapper` and `ctr plugins ls \| grep devmapper`, or set `kata_runtime: "kata-qemu"` |
| Sandbox pod stuck `Pending` | No node has both `ei-kata-eligible=true` and the `katacontainers.io/kata-runtime=true` label the RuntimeClass selects. `kubectl get nodes --show-labels` |
| `RuntimeClass 'kata-fc' was not created by kata-deploy` | That chart version has no fc shim. Check `kata_version` |
| Pod runs and `uname -r` matches the host | The pod landed on `runc`. Confirm `spec.runtimeClassName` |

```bash
kubectl logs -n kube-system -l name=kata-deploy --tail=100
kubectl describe pod -n agent-sandbox-system -l sandbox=kata
```

Remove the pool, the template, and the chart:

```bash
kubectl delete sandboxwarmpool kata-pool -n agent-sandbox-system
kubectl delete sandboxtemplate kata-sandbox-template -n agent-sandbox-system
helm uninstall kata-deploy -n kube-system
```

`kata-deploy` runs a cleanup hook on uninstall that removes the Kata files and the containerd configuration from each node.

To also remove the Firecracker thin pool from a node:

```bash
sudo systemctl disable --now containerd-devmapper.service
sudo rm -f /etc/systemd/system/containerd-devmapper.service /usr/local/bin/containerd-devmapper-setup.sh
sudo dmsetup remove containerd-thinpool
sudo rm -rf /var/lib/containerd/devmapper
# remove the "kata devmapper snapshotter (managed)" block from /etc/containerd/config.toml, then:
sudo systemctl restart containerd
```

OpenShell, which creates sandboxes through the same controller, is a separate guide: [OpenShell](openshell.md).

## Re-run

| Operation | What happens |
|---|---|
| `agent_sandbox_enabled: true` on a running stack | The playbook is idempotent. Manifests are re-applied with `kubectl apply` |
| Namespace already exists | Ansible creates it with `state: present` |
| Controller already running | Kubernetes applies the manifests. It does not create a second controller |
| Re-run | `./toolkit_manager.sh install agent-toolkit` applies the sandbox again. Running components stay |

To move to a new `agent_sandbox_version`, set the pin in `config.agent-toolkit.yaml` and re-run `install`, or [tear the stack down](../deployment/decommission.md) and install again. See [Upgrading](#upgrading).

## Upgrading

The version is pinned by `agent_sandbox_version` (default `v1.0.10`). Re-running `install` re-applies the manifests for that pin.

From an older pin (for example v0.5.0) to **v1.0.10**:

1. Back up existing resources:

   ```bash
   kubectl get sandboxes,sandboxclaims,sandboxtemplates,sandboxwarmpools -A -o yaml > backup.yaml
   ```

2. Set `agent_sandbox_version: "v1.0.10"` in `env/local/config.agent-toolkit.yaml` (or leave the default) and run `./toolkit_manager.sh install agent-toolkit`.
3. Upgrade the SDK: `pip install --upgrade k8s-agent-sandbox==1.0.10`.
4. Drop any client that relied on a static `ROUTER_AUTH_TOKEN`. The router now requires a Kubernetes service-account Bearer token (TokenReview).
5. Keep custom resources on `apiVersion: extensions.agents.x-k8s.io/v1beta1`.

For later versions, raise the pin, re-run `install`, and upgrade the SDK to match.

## Troubleshooting

| Symptom | Check |
|---|---|
| Pods not starting | `kubectl get pods -n agent-sandbox-system`, then `kubectl describe pod -n agent-sandbox-system <pod-name>` |
| Controller errors | `kubectl logs -n agent-sandbox-system -l app.kubernetes.io/name=agent-sandbox` |
| Router errors | `kubectl logs -n agent-sandbox-system -l app=sandbox-router` |
| Image pull errors | Test registry access: `kubectl run test --rm -it --restart=Never --image=registry.k8s.io/agent-sandbox/python-runtime-sandbox:v1.0.10 -- echo "success"`. For air-gapped clusters, mirror the images and set `agent_sandbox_router_image` and `agent_sandbox_python_runtime_image`. |
| SDK cannot connect | `kubectl get svc -n agent-sandbox-system sandbox-router-svc`, the `/healthz` check in [Verify](#3-verify), and `kubectl get networkpolicy -n agent-sandbox-system`. Router data-plane calls need a service-account Bearer token (TokenReview). |

## Next steps

- [Backend connection reference](backend-connections.md#agent-sandbox): in-cluster URL
- [Configuration](../configuration/configuration.md) and [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md)
- [Meet the Agent Toolkit](../overview/meet.md#toolkit-components)
- Upstream: [agent-sandbox docs](https://agent-sandbox.sigs.k8s.io/docs/), [v1.0.10 release notes](https://github.com/kubernetes-sigs/agent-sandbox/releases/tag/v1.0.10), [SDK README](https://github.com/kubernetes-sigs/agent-sandbox/tree/v1.0.10/clients/python/agentic-sandbox-client), [PyPI: k8s-agent-sandbox](https://pypi.org/project/k8s-agent-sandbox/1.0.10/)
