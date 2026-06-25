# local-llm

Self-hosted LLM stack: [Ollama](https://ollama.com/) for inference and
[Open WebUI](https://github.com/open-webui/open-webui) as the chat frontend.
Ollama runs on the GPU; Open WebUI connects to Ollama over the internal Docker network.

## Prerequisites

- Docker + Docker Compose v2
- An NVIDIA GPU with the
  [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
  installed (the `deploy.resources.reservations.devices` blocks require it)
- A `.env` file containing a secret key (already gitignored):

  ```sh
  echo "WEBUI_SECRET_KEY=$(openssl rand -hex 32)" > .env
  ```

## Usage

```sh
docker compose up -d        # start; open-webui waits until ollama is healthy
docker compose logs -f      # follow logs
docker compose down         # stop (named volumes persist)
```

Compose manages the `ollama` and `open-webui` named volumes automatically, so your models
and WebUI data survive restarts.

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

| Service     | URL                       | Notes                                    |
|-------------|---------------------------|------------------------------------------|
| Open WebUI  | `http://<host>:3000`      | Chat UI                                  |
| Ollama API  | `http://<host>:11434`     | No auth — exposed on the LAN. Trust your network. |

## Pulling and running models

The default model is pulled automatically (see above). To add more:

```sh
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama run llama3.2 "hello"
```

Models can also be pulled from within the Open WebUI interface.

## Notes

- Image tags are pinned for reproducibility; bump them deliberately when upgrading.
- Open WebUI uses the standard (non-CUDA) image. If you need GPU-accelerated
  RAG/embeddings inside Open WebUI, switch to the `:cuda` tag and add a `deploy`
  block for that service.
