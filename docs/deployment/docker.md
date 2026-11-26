# Docker Compose

[← Docs index](../README.md)

This path is separate from Toolkit Manager. Compose uses the in-tree `docker/` bundle (`bash docker/deploy.sh`). `toolkit_manager.sh` installs Kubernetes. For a first run with Toolkit Manager, use the [single-node quickstart](../getting-started/quickstart.md).

This path runs the agent stack as **containers on a single Linux host**, without Kubernetes. You still get a model engine (vLLM or OpenVINO™ Model Server), LiteLLM, Redis, and Postgres, started by Docker Compose rather than Helm. It is meant for a lab or development server where you do not want to provision a cluster. Production stays on Kubernetes. More engine detail: [`docker/README.md`](../../docker/README.md).

## Prerequisites

| Requirement | Minimum |
|---|---|
| Docker Engine | 24.0+ |
| Docker Compose | v2.20+ |
| RAM | 128 GB (256 GB if you start two or more LLMs) |
| Disk | 80 GB free |
| CPU | 96+ cores |
| Hugging Face token | Required for gated models (including the shipped default) |

## 1. Edit `docker/config.cfg`

The file shipped in the repo already sets a model. The default is gated Gemma, so put a Hugging Face token in `hugging_face_token=` before the first run (this path does **not** read `HF_TOKEN` from the shell):

```ini
hugging_face_token=hf_your_token_here
models=google/gemma-4-26B-A4B-it
# serving_engine=vllm          # default. Use openvino for OpenVINO™ Model Server
```

To use Qwen or several models instead (comma-separated, **no spaces**):

```ini
models=Qwen/Qwen2.5-Coder-14B-Instruct,meta-llama/Llama-3.1-8B-Instruct
```

## 2. Deploy from the repository root

```bash
bash docker/deploy.sh
```

Add `--observability` for Langfuse tracing. Other flags (`--full`, `--pull`, `--skip-build`, `--down`) are listed by `bash docker/deploy.sh --help`.

If containers are already running with the same model list, the script leaves them alone and only creates containers for models that are new. Secrets are written to `docker/vault.yml` and `docker/.env`. Do not commit those files.

## 3. Call a model

LiteLLM listens on `http://localhost:4000`. The LiteLLM UI is at `http://localhost:4000/ui`.

```bash
export LITELLM_MASTER_KEY=$(grep LITELLM_MASTER_KEY docker/.env | cut -d= -f2 | tr -d '"')

curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}"

curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model": "google/gemma-4-26B-A4B-it", "messages": [{"role": "user", "content": "Hello"}]}'
```

Use the full Hugging Face id as `"model"`: the same string you put in `models=` in `config.cfg`.

## What is listening

After a successful deploy, these ports are bound on the host. Credentials (database user, Redis password, LiteLLM master key) are in `docker/.env`.

| Service | Port | URL |
|---|---|---|
| Model engine (vLLM or OVMS) | 2080+ | `http://localhost:2080/v1` (next model gets 2081, …) |
| LiteLLM | 4000 | `http://localhost:4000` |
| Flowise (if enabled) | 3000 | `http://localhost:3000` |
| Langfuse (if enabled) | 3002 | `http://localhost:3002` |
| Postgres | 5432 | `localhost:5432` |
| Redis | 6379 | `localhost:6379` |

Credentials for Postgres and Redis are in `docker/.env`. Example:

```bash
source docker/.env
psql "postgresql://${PGVECTOR_USER}:${PGVECTOR_PASSWORD}@localhost:5432/${PGVECTOR_DB}"
redis-cli -h localhost -p 6379 -a "${REDIS_PASSWORD}"
```

## Models and engines

Put Hugging Face ids in `docker/config.cfg` (`models=`). Tuned flags and how to add a model: [`docker/README.md`](../../docker/README.md) and `docker/model-configs.yaml` (OpenVINO™: `docker/ovms-model-configs.yaml`). Do not edit generated `docker-compose.vllm.yml` or `litellm/config.yaml` by hand.

## Optional flags in `config.cfg`

```ini
serving_engine=vllm            # vllm | openvino
tool_call_parser=qwen3_coder   # none to disable (vLLM)
deploy_docker_fresh=off        # on = install Docker if missing
deploy_agenticai_plugin=off    # Flowise. Deprecated. Leave off.
deploy_observability=off       # Langfuse
http_proxy=
https_proxy=
no_proxy=localhost,127.0.0.1,.genai-gateway,.redis,.postgres,.flowise
```

`serving_engine=openvino` generates an OpenVINO™ Model Server compose file from `docker/ovms-model-configs.yaml` instead of vLLM. Parser values and more flags: [`docker/README.md`](../../docker/README.md).

## Troubleshooting

| Issue | What to try |
|---|---|
| Docker `start-limit-hit` | Reset the unit and start it again: `sudo systemctl reset-failed docker.service && sudo systemctl start docker.service`. |
| `apt-get` hangs | Set the proxy in `config.cfg` or `/etc/apt/apt.conf`. If a stuck `apt-get` is holding the lock, `sudo pkill -f apt-get`. |
| vLLM slow first start | Weights are downloading (often ~28 GB). Wait 5–10 minutes before treating it as a failure. |
| Out of memory | Reduce the number of Hugging Face ids in `models=`. Each extra LLM needs tens of gigabytes. |
| Port 2080 already taken | Put both models in `config.cfg` so ports increment, or `docker stop` / `docker rm` the old container. |
| Model missing in LiteLLM | Open the generated `docker/litellm/config.yaml` and confirm the model id is listed. |
| Generator fails | Check `yq --version` and that `yq eval '.' docker/model-configs.yaml` parses. |

```bash
docker compose -f docker/docker-compose.vllm.yml logs vllm-<name>
```

## Stop the stack

```bash
# Keep volumes
docker compose -f docker/docker-compose.yml -f docker/docker-compose.vllm.yml down

# Delete data too (passwords are regenerated on the next deploy)
bash docker/deploy.sh --down
```

## Related pages

- [Docs index](../README.md)
- [Single-node quickstart](../getting-started/quickstart.md): one-host Kubernetes install with Toolkit Manager
- [Configuration](../configuration/configuration.md): YAML for the Kubernetes install
- [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md#docker-compose)
- [Meet the Agent Toolkit](../overview/meet.md)
