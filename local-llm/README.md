# local-llm

Self-hosted LLM stack: [Ollama](https://ollama.com/) for GPU inference, exposed
only over the [Tailscale](https://tailscale.com/) tailnet as `local-llm`.

Ollama shares the network namespace of a Tailscale sidecar
(`network_mode: service:tailscale`), the same pattern used by `hermes-agent/`.
There is **no LAN port** — the API is reachable only from devices on your tailnet.

## Prerequisites

- Docker + Docker Compose v2
- An NVIDIA GPU with the
  [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
  installed (the `deploy.resources.reservations.devices` block requires it)
- A Tailscale account, with HTTPS/MagicDNS enabled in the tailnet admin (needed
  for the `https://local-llm` endpoint)
- A `.env` file containing a Tailscale auth key (already gitignored):

  ```sh
  echo "TS_AUTHKEY=tskey-..." > .env
  ```

  Generate a reusable or ephemeral key at
  <https://login.tailscale.com/admin/settings/keys>.

## Usage

```sh
docker compose up -d        # start; ollama waits until tailscale is healthy
docker compose logs -f      # follow logs (tailscale auth, model warm-up)
docker compose down         # stop (named volumes persist)
```

Compose manages the `ollama` and `tailscale-state` named volumes automatically,
so your models and the Tailscale node identity survive restarts.

## Default model

On first `docker compose up`, the `ollama-init` service automatically pulls and
warms up a default model:

    hf.co/yuxinlu1/gemma-4-12B-agentic-fable5-composer2.5-v2-3.5x-tau2-GGUF:Q3_K_M

This is a one-shot container (`restart: "no"`) that exits after the model is
loaded. Subsequent starts skip the pull if the model is already present in the
`ollama` volume.

`OLLAMA_KEEP_ALIVE=-1` is set on the `ollama` service so that loaded models
remain in GPU memory indefinitely (no idle unloading).

## Access

The Ollama API is reachable from any device on your tailnet:

| Endpoint                   | Notes                                             |
|----------------------------|---------------------------------------------------|
| `http://local-llm:11434`   | Native Ollama API (MagicDNS hostname)             |
| `https://local-llm`        | Same API over HTTPS 443, terminated by `tailscale serve` (see `serve.json`) |

Quick check from another tailnet device:

```sh
curl http://local-llm:11434/api/tags
curl https://local-llm/api/tags
```

## Pulling and running models

The default model is pulled automatically (see above). To add more:

```sh
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama run llama3.2 "hello"
```

## Notes

- Image tags are pinned for reproducibility; bump them deliberately when upgrading.
- `serve.json` controls the `tailscale serve` config (HTTPS 443 → `127.0.0.1:11434`).
  `${TS_CERT_DOMAIN}` is expanded by Tailscale to this node's cert domain at runtime.
- Because `ollama` uses `network_mode: service:tailscale`, it cannot publish its
  own host ports; the `ollama-init` helper reaches Ollama via the `tailscale`
  service name on the compose network.
