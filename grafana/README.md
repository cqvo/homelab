# grafana

[Grafana](https://grafana.com/) fronted by a [Tailscale](https://tailscale.com/)
sidecar so the dashboard is reachable on the tailnet at its own MagicDNS name —
**no port required**.

The `tailscale` container owns the tailnet identity (`hostname: grafana`) and
runs [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve), which
terminates HTTPS on 443 and reverse-proxies to Grafana. Grafana shares the
sidecar's network namespace (`network_mode: service:tailscale`) and listens on
`127.0.0.1:3000`. No host port is published, so access is tailnet-only.

## Prerequisites

- Docker + Docker Compose v2
- A Tailscale account with **MagicDNS and HTTPS certificates enabled** in the
  [admin console](https://login.tailscale.com/admin/dns) (Serve over HTTPS
  requires this).
- A `.env` file (already gitignored) — copy the example and fill it in:

  ```sh
  cp .env.example .env
  # TS_AUTHKEY  -> https://login.tailscale.com/admin/settings/keys
  # TS_TAILNET  -> your tailnet domain, e.g. tailnet-abcd.ts.net
  ```

## Usage

```sh
docker compose up -d        # grafana waits until the tailscale node is healthy
docker compose logs -f      # follow logs
docker compose down         # stop (named volumes persist)
```

## Access

| Service | URL                                | Notes                                   |
|---------|------------------------------------|-----------------------------------------|
| Grafana | `https://grafana.<tailnet>.ts.net` | Tailnet-only, valid TLS cert, no port.  |

Default login is `admin` / `admin`; Grafana prompts you to change it on first
sign-in. Dashboards, data sources, and settings persist in the `grafana-data`
named volume across restarts.

## Notes

- Image tags are pinned for reproducibility; bump them deliberately when
  upgrading.
- `${TS_CERT_DOMAIN}` in `serve.json` is substituted at runtime with the node's
  FQDN, so the Serve config is portable.
- `TS_EXTRA_ARGS=--reset` keeps the running Serve config in sync with
  `serve.json` on every restart.
