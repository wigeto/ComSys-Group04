# Group 04 — Self-Hosting Middleware

DNS and HTTPS reverse proxy for the Communication Systems Lab, with optional
Uptime Kuma monitoring and dtop container management.

## Services

Unbound serves DNS records for `comsyslab.csg.uzh.ch` and its application
hostnames, all pointing to the middleware VM at `192.41.136.194`. It answers
local records only; recursive queries for other domains are refused.

Caddy handles HTTPS and routes requests to the application VMs:

| Hostname | Backend |
| --- | --- |
| `onlyoffice.comsyslab.csg.uzh.ch` | `192.168.100.5:8081` |
| `nextcloud.comsyslab.csg.uzh.ch` | `192.168.100.5:80` |
| `immich.comsyslab.csg.uzh.ch` | `192.168.100.6:2283` |
| `media.comsyslab.csg.uzh.ch` | `192.168.100.7:8096` |

The root site redirects `/onlyoffice`, `/nextcloud`, `/immich`, and `/media` to
the corresponding HTTPS site. Each application hostname serves `/caddyHealth`
with HTTP 200 to check Caddy independently of the backend.

## Deployment

Run from this directory on the middleware VM with Docker Engine and the Compose
plugin installed. The host needs both `192.168.100.8` and `192.41.136.194`,
available ports `53`, `80`, and `443`, and access to the application VMs.
The Docker subnet `172.18.0.0/16` must not overlap existing networks.

```sh
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=100 unbound caddy
```

Configure DNS in `unbound.conf` and proxy routes in `Caddyfile`. Caddy manages
HTTPS certificates automatically; issuance requires suitable DNS and network
reachability for certificate validation. No `.env` file is required.

DNS and HTTP/HTTPS are published on both host addresses. UDP port 443 for
HTTP/3 is published only on `192.41.136.194`.

To stop the stack:

```sh
docker compose down
```

## Verification

From a client with access to the middleware VM:

```sh
dig @192.168.100.8 nextcloud.comsyslab.csg.uzh.ch A
curl -i --resolve nextcloud.comsyslab.csg.uzh.ch:443:192.41.136.194 \
  https://nextcloud.comsyslab.csg.uzh.ch/caddyHealth
```

Expect DNS to return `192.41.136.194` and the HTTPS check to return HTTP 200.
Open the application URLs separately to verify backend availability.

## Monitoring

```sh
docker compose -f dockerMonitor/docker-compose.yaml up -d
docker compose -f dtop/docker-compose.yaml up -d
```

For Uptime Kuma, open an SSH tunnel from your computer using your configured
`comsys-grp-4` alias, then visit `http://localhost:3001`:

```sh
ssh -N -L 3001:127.0.0.1:3001 comsys-grp-4
```

Kuma also publishes port `3001` on all host interfaces. Its Compose file still
references an old step-ca certificate path without mounting the certificate;
review that setting if custom CA trust is needed.

For dtop, run `docker attach dtop` on the VM. Detach with `Ctrl-p`, then `Ctrl-q`.
Both tools mount the host Docker socket and can manage containers.

## Persistent data

Back up these host directories separately from the configuration:

- `data/caddy_data/` — certificates and Caddy state.
- `data/caddy_config/` — Caddy configuration state.
- `/home/comsys/dockerMonitor/data` — live Uptime Kuma data.

The tracked files under `dockerMonitor/data` are not necessarily Kuma's live
database; Compose uses the absolute path above. No backup or restore automation
is included.
