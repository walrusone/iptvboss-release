# IPTVBoss XC Server container

This repository publishes the headless IPTVBoss XC Server for amd64 and arm64. The current pre-release image is:

```text
ghcr.io/walrusone/iptvboss-release
```

Version tags such as `3.11.21` are intended for pinned deployments. The moving `latest` tag follows the newest container published in the current channel.

## Standalone Compose setup

Download these files into one directory:

- `compose.yaml`
- `env.example`

Copy the environment example and edit it:

```sh
cp env.example .env
nano .env
```

The default configuration runs only IPTVBoss and publishes unencrypted HTTP on port `8001` for access from a trusted local network:

```env
IPTVBOSS_HOST_IP=0.0.0.0
IPTVBOSS_HOST_PORT=8001
IPTVBOSS_XC_PORT=8001
IPTVBOSS_XC_BEHIND_HTTPS_PROXY=false
IPTVBOSS_HTTPS_ONLY=false
```

Do not forward port `8001` from the Internet. Use a private host address instead of `0.0.0.0` when IPTVBoss should listen on only one network interface.

Start the stack:

```sh
docker compose config
docker compose pull
docker compose up --detach
docker compose ps
docker compose logs --follow
```

Open `http://server-private-ip:8001/boss.php`. On a new data volume, the first visitor creates the administrator and completes bootstrap.

`IPTVBOSS_XC_PORT` controls the listener inside the container. It defaults to
`8001`. The existing `IPTVBOSS_HOST_PORT` controls the host-side published
port; the Compose template maps that host port to the configured XC listener:

```env
IPTVBOSS_XC_PORT=9000
```

For non-Compose launches, use `IPTVBOSS_XC_PORT` or `-xc-port 9000`; the CLI
option takes precedence over the environment variable. When neither is
supplied, the persisted XC Server port is used.

## Bundle Caddy in the same Compose file

For public HTTPS, download `Caddyfile` beside `compose.yaml`. Point a public hostname's DNS records to the Docker host and ensure inbound TCP ports `80` and `443` reach it. UDP port `443` is optional for HTTP/3.

Change or add these values in `.env`:

```env
IPTVBOSS_DOMAIN=boss.example.com
IPTVBOSS_HOST_IP=127.0.0.1
IPTVBOSS_XC_BEHIND_HTTPS_PROXY=true
```

Add the `caddy` service under `services` in `compose.yaml`, aligned with the existing `iptvboss` service:

```yaml
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    environment:
      IPTVBOSS_DOMAIN: "${IPTVBOSS_DOMAIN:-boss.domain.com}"
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      iptvboss:
        condition: service_healthy
```

Add the Caddy volumes to the existing `volumes` section:

```yaml
volumes:
  iptvboss-data:
  caddy-data:
  caddy-config:
```

Validate and apply the expanded stack:

```sh
docker compose config
docker compose pull
docker compose up --detach
docker compose ps
```

Both services should be running. Open `https://boss.example.com/boss.php` after Caddy obtains its certificate. Binding IPTVBoss to host loopback prevents clients from bypassing Caddy over unencrypted HTTP.

## Use an existing proxy on the Docker host

If an HTTPS reverse proxy already runs directly on the Docker host, configure it to use this upstream:

```text
http://127.0.0.1:8001
```

The proxy must send `X-Forwarded-Proto: https`. Keep proxy mode enabled:

```env
IPTVBOSS_XC_BEHIND_HTTPS_PROXY=true
IPTVBOSS_HTTPS_ONLY=false
IPTVBOSS_HOST_IP=127.0.0.1
```

A proxy in another container has separate loopback networking. Attach it to a shared user-defined network and use `http://iptvboss:8001` by default, replacing `8001` with the configured `IPTVBOSS_XC_PORT` when needed, or publish the backend on a private host address protected by a firewall. Do not expose the direct HTTP listener to the Internet.

## Data and backups

IPTVBoss databases, configuration, generated XC files, caches, and logs are stored in the named `iptvboss-data` volume. When bundled Caddy is added, it uses separate `caddy-data` and `caddy-config` volumes.

Stop IPTVBoss briefly and copy its data to a dated directory before upgrades:

```sh
mkdir -p backups/2026-08-23
docker compose stop iptvboss
docker compose cp --archive iptvboss:/data/. ./backups/2026-08-23/
docker compose start iptvboss
```

Do not run `docker compose down --volumes` unless all persistent data managed by the stack should be deleted.

## Update or pin a version

The image repository and tag are configured independently so a future release channel can be selected without editing `compose.yaml`:

```env
IPTVBOSS_IMAGE=ghcr.io/walrusone/iptvboss-release
IPTVBOSS_TAG=latest
```

For a controlled deployment, replace the tag with an exact tested version. After making a backup:

```sh
docker compose pull
docker compose up --detach
```

If a rollback is necessary, restore the matching pre-upgrade data backup before starting an older image when its database schema may differ.

## Advanced settings

The container defaults to non-root UID/GID `10001:10001`. Change `IPTVBOSS_UID` and `IPTVBOSS_GID` only when a bind-mounted host directory requires another owner.

When the published backend can be reached by untrusted peers, restrict forwarded headers to the real proxy peer addresses or CIDRs:

```env
IPTVBOSS_XC_TRUSTED_PROXIES=172.20.0.0/16,127.0.0.1/32
```

Docker address translation can make the peer appear as a bridge gateway. Use the rejected-peer address reported by IPTVBoss and include every trusted proxy hop.

Direct HTTPS is available for installations that cannot use a reverse proxy. Disable proxy mode, enable HTTPS, and mount a valid PKCS#12 store at `/data/keystore.p12`:

```env
IPTVBOSS_XC_BEHIND_HTTPS_PROXY=false
IPTVBOSS_HTTPS_ONLY=true
IPTVBOSS_XC_KEYSTORE_PASSWORD=replace-with-keystore-password
IPTVBOSS_HOST_IP=0.0.0.0
```

The password unlocks the TLS private-key store only. It does not encrypt IPTVBoss data. Proxy mode takes precedence and does not use the keystore.

## Pull without Compose

The package is public and can be pulled without a GitHub login:

```sh
docker pull ghcr.io/walrusone/iptvboss-release:latest
```
