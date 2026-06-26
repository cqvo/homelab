# home-assistant

[Home Assistant](https://www.home-assistant.io/) fronted by a
[Tailscale](https://tailscale.com/) sidecar so the dashboard is reachable on the
tailnet at its own MagicDNS name — **no port required**.

The `tailscale` container owns the tailnet identity (`hostname: home-assistant`)
and runs [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve), which
terminates HTTPS on 443 and reverse-proxies to Home Assistant. Home Assistant
shares the sidecar's network namespace (`network_mode: service:tailscale`) and
listens on `127.0.0.1:8123`. No host port is published, so access is
tailnet-only.

## Prerequisites

- Docker + Docker Compose v2
- A Tailscale account with **MagicDNS and HTTPS certificates enabled** in the
  [admin console](https://login.tailscale.com/admin/dns) (Serve over HTTPS
  requires this).
- A `.env` file (already gitignored) — copy the example and fill it in:

  ```sh
  cp .env.example .env
  # TS_AUTHKEY  -> https://login.tailscale.com/admin/settings/keys
  # TZ          -> your timezone, e.g. America/Los_Angeles
  ```

## Usage

```sh
docker compose up -d        # home assistant waits until the tailscale node is healthy
docker compose logs -f      # follow logs
docker compose down         # stop (config persists in ./config)
```

## Access

| Service        | URL                                       | Notes                                  |
|----------------|-------------------------------------------|----------------------------------------|
| Home Assistant | `https://home-assistant.<tailnet>.ts.net` | Tailnet-only, valid TLS cert, no port. |

On first visit Home Assistant runs its onboarding wizard to create your owner
account. Configuration lives in the bind-mounted `./config` directory and
persists across restarts.

## Notes

- Home Assistant sits behind the Serve reverse proxy, so `config/configuration.yaml`
  ships with an `http:` block that trusts the loopback proxy
  (`use_x_forwarded_for` + `trusted_proxies`). Without it, HA rejects every
  request with `400: Bad Request`.
- `./config` is bind-mounted so the config is editable from the host and lives
  outside a named volume. Everything Home Assistant generates there is
  gitignored except the seed `configuration.yaml`.
- Image tags are pinned for reproducibility; bump them deliberately when
  upgrading.
- `${TS_CERT_DOMAIN}` in `serve.json` is substituted at runtime with the node's
  FQDN, so the Serve config is portable.
- `TS_EXTRA_ARGS=--reset` keeps the running Serve config in sync with
  `serve.json` on every restart.
