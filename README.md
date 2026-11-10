# Group 04 — Self-Hosting Middleware

Middleware for the Communication Systems Lab: internal DNS, an HTTPS reverse
proxy, and a private certificate authority for the other teams' services.
All middleware components run in Docker containers managed through Docker Compose.


## Architecture

Clients use **Unbound** at `192.168.100.8:53` to resolve service names to the
middleware VM. They then connect to **Caddy** over HTTPS. Caddy forwards each
request to the corresponding application VM and obtains certificates from
**step-ca** through ACME.

All three components run in Docker Compose with persistent bind mounts and
`restart: unless-stopped`. The Docker subnet is fixed at `172.18.0.0/16` so
Unbound's access rules remain consistent when containers are recreated.

## Technology decisions

| Component | Choice | Why we chose it |
| --- | --- | --- |
| DNS | Unbound | Lightweight recursive resolver with caching, local DNS records, and native DNSSEC support. Fits a small lab with known VM addresses. |
| Reverse proxy | Caddy | Concise configuration, automatic HTTPS and certificate renewal, and built-in WebSocket proxying. Reduces manual certificate management. |
| Certificate authority | step-ca | Lightweight private CA with ACME support, allowing Caddy to obtain and renew certificates automatically. Keeps certificate issuance under our control. |
| Deployment | Docker Compose | Describes services, networks, and storage together, making the deployment easier to reproduce and maintain. |

These components are open source and can run without proprietary hosted DNS,
proxy, or certificate services.

We considered dnsmasq and CoreDNS. dnsmasq's combined DNS/DHCP functionality is
unnecessary for our current scope; Unbound's recursive resolution is a better
fit. CoreDNS offers a useful plugin architecture for dynamic service discovery
and Kubernetes, but that flexibility is not currently needed. It could later
complement Unbound for internal DNS while Unbound retains recursive resolution.

## Certificates and security

Caddy explicitly uses step-ca's ACME endpoint at
`https://step-ca:9000/acme/acme/directory`. It automatically renews service
certificates before expiry; no separate renewal cron job is required. Renewal
depends on step-ca being available and able to resolve and reach the service
hostnames for ACME validation. Caddy's state persists in `data/caddy_data/`.

CA private keys are encrypted, stored outside Git, and protected by restricted
filesystem permissions. Of the CA's files, only the public root certificate is
mounted into Caddy.

Clients must be connected to the lab network, use our DNS server, and trust
`data/step-ca/certs/root_ca.crt`. Public DNS does not direct users to this setup.

## Current status

DNS resolution and initial HTTPS certificate issuance through step-ca have been
verified on the server. Automatic renewal still needs to be observed.
`service1.pilot.internal` through `service4.pilot.internal` and backend address
`192.0.2.10:80` are placeholders pending coordination with the other teams.

Backups and disaster recovery remain to be implemented.

## References

- [Unbound](https://www.nlnetlabs.nl/projects/unbound/about/)
- [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [step-ca](https://smallstep.com/docs/step-ca/)
