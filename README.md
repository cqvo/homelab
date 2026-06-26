# homelab

Personal infrastructure-as-code repository. Each subdirectory is a self-contained
service stack managed with Docker Compose.

## Stacks

| Stack                      | Description                                          |
|----------------------------|------------------------------------------------------|
| [local-llm](./local-llm/) | Ollama + Open WebUI — self-hosted LLM inference       |
| [grafana](./grafana/)     | Grafana + Tailscale sidecar — tailnet-only dashboards |

## Prerequisites

- Docker + Docker Compose v2
- Service-specific requirements are documented in each stack's README

## Repository layout

```
.
├── local-llm/          # Self-hosted LLM stack
│   ├── .env.example
│   ├── docker-compose.yaml
│   └── README.md
├── grafana/            # Grafana behind a Tailscale sidecar
│   ├── .env.example
│   ├── docker-compose.yaml
│   ├── serve.json
│   └── README.md
└── README.md           # This file
```
