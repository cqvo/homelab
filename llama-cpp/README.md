# llama-cpp

Self-hosted [llama.cpp](https://github.com/ggml-org/llama.cpp) server, fronted by
[Tailscale](https://tailscale.com/) for secure remote access. The `llama-server`
container shares the `tailscale` container's network namespace and is reachable over
the tailnet on HTTPS `:443`, proxied to `127.0.0.1:8080` (see `serve.json`).

The stack runs CPU-only by default and switches to an NVIDIA GPU entirely through `.env`.

## Prerequisites

- Docker + Docker Compose v2
- A `.env` file (gitignored) — copy `.env.example` and fill in at least `TS_AUTHKEY`,
  `TS_TAILNET`, and `MODEL_NAME`.

## Usage

```sh
docker compose up -d        # start; llama-server waits until tailscale is healthy
docker compose logs -f      # follow logs
docker compose down         # stop (named volumes persist)
```

Models downloaded via `--hf-repo` are cached in the `llama-cpp` named volume
(`LLAMA_CACHE=/models`), so they survive restarts. The first start can take a while
(model download + load) — the healthcheck `start_period` is 30m to cover this.

## CPU vs GPU (`.env`-driven)

CPU is the default — leave the GPU variables unset. To use an NVIDIA GPU, set these in
`.env`:

```dotenv
# Path separator is ';' on Windows, ':' on Unix.
COMPOSE_FILE=docker-compose.yaml;docker-compose.gpu.yaml
LLAMA_IMAGE_TAG=server-cuda
LLAMA_N_GPU_LAYERS=99           # 99 = offload all layers; lower for partial offload
```

- `COMPOSE_FILE` merges `docker-compose.gpu.yaml`, which adds the NVIDIA device
  reservation. Without it, the reservation isn't applied (and won't break CPU-only hosts).
- `LLAMA_IMAGE_TAG=server-cuda` selects the CUDA build of the image.
- `LLAMA_N_GPU_LAYERS` is passed as `--n-gpu-layers`; it defaults to `0` (CPU).

## Windows (Docker Desktop / WSL2)

- Install a recent **NVIDIA Windows driver** with WSL2 CUDA support — confirm
  `nvidia-smi` works inside WSL2.
- Run **Docker Desktop on the WSL2 backend**. GPU passthrough works out of the box; the
  NVIDIA Container Toolkit is bundled (no separate install like on native Linux).
- Set the GPU variables above. On Windows the `COMPOSE_FILE` separator is `;` (use that,
  or set `COMPOSE_PATH_SEPARATOR=:` to keep the `:` form).
- Keep `.env` line endings as **LF** — CRLF can corrupt values and the `COMPOSE_FILE` list.
- **Tailscale needs `/dev/net/tun`**, normally present on Docker Desktop/WSL2. If the
  tailscale container errors on tun, set `TS_USERSPACE=true` in `.env` as a fallback
  (`tailscale serve` proxying to `127.0.0.1:8080` still works in userspace mode).

### VRAM sizing

The default 12B `Q4_K_M` model needs roughly **8 GB VRAM** for full offload
(`LLAMA_N_GPU_LAYERS=99`). If the GPU has less, lower the layer count for a partial
offload, or pick a smaller quant/model via `MODEL_NAME`.

## Verifying GPU use

```sh
# 1. GPU visible to Docker
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi

# 2. Confirm the merge + variable substitution
docker compose config        # expect server-cuda image, --n-gpu-layers 99, deploy.resources

# 3. Bring up and watch for CUDA init
docker compose up -d
docker compose logs -f llama-server   # look for ggml_cuda_init / "offloaded N/N layers to GPU"

# 4. Confirm VRAM is in use
nvidia-smi                   # llama-server process holding VRAM
```

## Notes

- Image tags are pinned for reproducibility; bump them deliberately when upgrading.
- `serve.json` defines the Tailscale HTTPS proxy (`:443` → `127.0.0.1:8080`).
