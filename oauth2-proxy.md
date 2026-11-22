---
title: oauth2-proxy (SSO auth gateway)
description: oauth2-proxy SSO gateway with Caddy forward_auth and Keycloak, using Docker Compose.
published: true
date: 2026-11-22T06:33:42.727Z
tags: docker, docker-compose, homelab, identity, sso, caddy, keycloak
editor: markdown
dateCreated: 2026-11-22T06:33:42.727Z
---

# oauth2-proxy (SSO auth gateway)

> oauth2-proxy as a single-sign-on gateway in front of your apps, with Caddy `forward_auth` and Keycloak as the identity provider, using Docker Compose.
{.is-info}

_Last updated: Nov 2026_

## Overview

oauth2-proxy sits in front of apps that have no login of their own (or whose login you do not want to expose). Users sign in once at [Keycloak](/keycloak). Your reverse proxy asks oauth2-proxy whether the request is allowed, then either passes it through or redirects the browser to sign in.

**Why you'd want it:** One sign-in for admin tools and dashboards, without giving each app its own password.

| Item | Value |
|---|---|
| Stack folder | `/opt/stacks/oauth2-proxy` |
| Category | Identity and security |
| Main image | `quay.io/oauth2-proxy/oauth2-proxy:v7.12.1` |
| Services | 1 |
| Web UI | https://auth.<YOUR_DOMAIN> |
| Official docs | https://oauth2-proxy.github.io/oauth2-proxy/ |

## Requirements

- A Linux host with Docker Engine and the Docker Compose plugin (see [Docker](/docker))
- The shared folder layout and `.env`/`.secret` files from [Docker Folders](/dockerfolders) and [Environment](/environment)
- A working [Keycloak](/keycloak) realm and a confidential OIDC client for oauth2-proxy
- A reverse proxy with TLS. This guide uses Caddy `forward_auth` as the example

## Compose

Folder layout:

```text
/opt/stacks/
├── _shared/
│   ├── .env        # shared, non-secret settings
│   └── .secret     # passwords, tokens and keys (chmod 600, never commit)
└── oauth2-proxy/
    └── compose.yaml
/opt/appdata/oauth2-proxy/   # persistent application data
```

`/opt/stacks/oauth2-proxy/compose.yaml`:

```yaml
---
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.12.1
    container_name: oauth2-proxy
    hostname: oauth2-proxy
    env_file:
      - /opt/stacks/_shared/.env
      - /opt/stacks/_shared/.secret
    command:
      - --http-address=0.0.0.0:4180
      - --metrics-address=0.0.0.0:44180
      - --reverse-proxy=true
      - --upstream=static://202
      - --provider=oidc
      - --provider-display-name=Keycloak
      - --oidc-issuer-url=https://keycloak.<YOUR_DOMAIN>/realms/<REALM>
      - --redirect-url=https://auth.<YOUR_DOMAIN>/oauth2/callback
      - --code-challenge-method=S256
      - --email-domain=*
      - --cookie-secure=true
      - --cookie-httponly=true
      - --cookie-samesite=lax
      - --session-cookie-minimal=true
      - --whitelist-domain=.<YOUR_DOMAIN>
      - --cookie-domain=.<YOUR_DOMAIN>
      - --set-xauthrequest=true
      - --pass-access-token=false
    environment:
      - OAUTH2_PROXY_CLIENT_ID=${OAUTH2_PROXY_CLIENT_ID:?err}
      - OAUTH2_PROXY_CLIENT_SECRET=${OAUTH2_PROXY_CLIENT_SECRET:?err}
      - OAUTH2_PROXY_COOKIE_SECRET=${OAUTH2_PROXY_COOKIE_SECRET:?err}
    logging:
      driver: json-file
      options:
        max-file: ${DOCKERLOGGING_MAXFILE:-10}
        max-size: ${DOCKERLOGGING_MAXSIZE:-200k}
    ports:
      - 4180:4180
    restart: unless-stopped
```

## Caddy `forward_auth` example

TLS and HTTP routing stay on Caddy. oauth2-proxy only answers "is this request signed in?". Replace every `<PLACEHOLDER>`.

```caddy
# Snippet you can import into site blocks that should require SSO.
(sso_forward_auth) {
	forward_auth oauth2-proxy:4180 {
		uri /oauth2/auth
		copy_headers X-Auth-Request-User X-Auth-Request-Email
	}
}

# The sign-in and callback host for oauth2-proxy itself.
auth.<YOUR_DOMAIN> {
	reverse_proxy oauth2-proxy:4180
}

# Example protected app. Repeat this pattern for each tool you want behind SSO.
app.<YOUR_DOMAIN> {
	import sso_forward_auth
	reverse_proxy <APP_CONTAINER>:<APP_PORT>
}
```

If Caddy and oauth2-proxy do not share a Docker network, point `forward_auth` at `http://127.0.0.1:4180` instead of the container name.

## Environment variables

Compose reads these from the shared files with `--env-file`. Replace every `<PLACEHOLDER>` with your own value.

`/opt/stacks/_shared/.env`:

```ini
PUID=1000
PGID=1000
TZ=<TZ>
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k
```

`/opt/stacks/_shared/.secret` (chmod 600):

```ini
OAUTH2_PROXY_CLIENT_ID=<CLIENT_ID>
OAUTH2_PROXY_CLIENT_SECRET=<CLIENT_SECRET>
OAUTH2_PROXY_COOKIE_SECRET=<COOKIE_SECRET>
```

Generate the cookie secret with `openssl rand -base64 32` and paste the result. Never commit this file.

| Variable | File | Purpose | Example |
|---|---|---|---|
| PUID | .env | User ID the container runs as (see Docker Folders) | 1000 |
| PGID | .env | Group ID the container runs as | 1000 |
| TZ | .env | Time zone in tz database format, e.g. America/New_York | <TZ> |
| DOCKERLOGGING_MAXFILE | .env | Number of rotated container log files to keep | 10 |
| DOCKERLOGGING_MAXSIZE | .env | Maximum size of each container log file | 200k |
| OAUTH2_PROXY_CLIENT_ID | .secret | Keycloak confidential client ID | <CLIENT_ID> |
| OAUTH2_PROXY_CLIENT_SECRET | .secret | Keycloak confidential client secret | <CLIENT_SECRET> |
| OAUTH2_PROXY_COOKIE_SECRET | .secret | 16, 24 or 32-byte cookie encryption key | <COOKIE_SECRET> |

## Ports

| Host port | Container port | Protocol | Service | Purpose |
|---|---|---|---|---|
| 4180 | 4180 | tcp | oauth2-proxy | Auth gateway and `/oauth2/*` endpoints |

## Volumes

This service is stateless when the session is stored in the cookie. No app-data volume is required.

## Resource limits

No limits are set. This app is light, so add `deploy.resources.limits` only if you need to cap it.

## Setup

1. Create the folders:
   ```bash
   sudo mkdir -p /opt/stacks/oauth2-proxy /opt/appdata/oauth2-proxy
   sudo chown -R $USER: /opt/stacks/oauth2-proxy
   ```
2. Add the variables above to `/opt/stacks/_shared/.env` and `/opt/stacks/_shared/.secret` (`chmod 600 .secret`).
3. Save the compose file as `/opt/stacks/oauth2-proxy/compose.yaml`.
4. Check the configuration, then start the stack:
   ```bash
   cd /opt/stacks/oauth2-proxy
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret config --quiet
   docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
   ```
5. Watch the first start: `docker compose logs -f`.

6. In Keycloak, create a confidential client. Set the redirect URI to `https://auth.<YOUR_DOMAIN>/oauth2/callback` and the web origin to `https://auth.<YOUR_DOMAIN>`.
7. Put the client ID, client secret and cookie secret in `.secret`.
8. Proxy `auth.<YOUR_DOMAIN>` to port 4180, then add `forward_auth` to each site block you want to protect, as in the Caddy example.
9. Open `https://auth.<YOUR_DOMAIN>` and sign in with a Keycloak user to confirm the redirect loop ends at your app.

> Tip: create a small wrapper such as `alias dc='docker compose --env-file /opt/stacks/_shared/.env --env-file /opt/stacks/_shared/.secret'` so every stack uses the same env files.
{.is-success}
## Updating

```bash
cd /opt/stacks/oauth2-proxy
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret pull
docker compose --env-file ../_shared/.env --env-file ../_shared/.secret up -d
docker image prune -f
```

Back up `/opt/appdata/oauth2-proxy` before major upgrades. [Watchtower](/watchtower) can automate updates for stateless apps.

## Troubleshooting

- Check status and logs: `docker compose ps` and `docker compose logs --tail=100`.
- **"redirect_uri mismatch":** the Keycloak client redirect URI must match `--redirect-url` exactly, including `https`.
- **Redirect loop after login:** Caddy must pass `X-Forwarded-Proto` and `X-Forwarded-Host`, and `--cookie-secure` requires HTTPS.
- **Cookie not sent to apps on other names:** `--cookie-domain` and `--whitelist-domain` should be `.<YOUR_DOMAIN>` (leading dot).
- **A variable is empty or reported missing (`:?err`):** run the `config` command above to see which variable is not set in your env files.

## Related

- [Docker Stack Catalog](/docker-stacks)
- [Keycloak](/keycloak)
- [IT-Tools](/it-tools)
- [Dozzle](/dozzle)
- [Adminer](/adminer)

---
*Educational example. Review every path, port and credential before you deploy, and never publish your `.secret` file.*
