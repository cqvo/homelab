# local-llm

Self-hosted LLM stack: [Ollama](https://ollama.com/) for inference and
[Open WebUI](https://github.com/open-webui/open-webui) as the chat frontend, both GPU-accelerated.

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

## Access

| Service     | URL                       | Notes                                    |
|-------------|---------------------------|------------------------------------------|
| Open WebUI  | `http://<host>:3000`      | Chat UI                                  |
| Ollama API  | `http://<host>:11434`     | No auth — exposed on the LAN. Trust your network. |

## Pulling and running models

```sh
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama run llama3.2 "hello"
```

Models can also be pulled from within the Open WebUI interface.

## Notes

- Image tags are pinned for reproducibility; bump them deliberately when upgrading.
- The Open WebUI `:cuda` image only needs the GPU for local RAG/embeddings. If you rely on
  Ollama for everything, you can drop its `deploy` block and switch to the non-cuda image.
