# Deploy Stack (Pre-built Images)

This folder contains a `compose.yaml` that uses only pre-built images — no local
`build:` context. Use it when your deployment tool cannot build images at deploy time
(Portainer CE, a remote Docker host, any GitOps runner that expects a registry image).

The root [compose.yaml](../compose.yaml) builds `cryanc/carl` and `browservice` from
source and is meant for local development. Here, both are replaced by images published
to GHCR by GitHub Actions:

```text
ghcr.io/tommiec/cryanc-carl:latest
ghcr.io/tommiec/browservice:latest
```

These images must be public for unauthenticated pulls. If you fork this repository
and publish your own images, update the image names in this compose file.

## Files

- `compose.yaml` — stack definition using pre-built images only.
- `.env.example` — template for the `.env` file; copy and fill in `PROXY_HOSTNAME`.

This generic deploy compose uses Docker named volumes for AdGuard Home data:

```yaml
adguard_work:
adguard_conf:
```

That keeps this repository portable. A NAS-specific GitOps repository may
replace those named volumes with host bind mounts if it wants all runtime data
under one backup/cleanup root.

## Setup

1. Copy `.env.example` to `.env` alongside `compose.yaml` and set `PROXY_HOSTNAME`.
2. Deploy with your tool of choice:
   - **Plain Docker Compose**: `docker compose up -d`
   - **Portainer CE**: point the stack at this `compose.yaml` and supply `.env` as the
     environment file.
   - **GitOps runner**: commit `compose.yaml`; keep `.env` out of git and supply it as
     a secret or side-loaded file.
3. Configure the browser's proxy to `<NAS-IP>` and the relevant port (see the root
   [README](../README.md#ports)).

Optional: to use the included AdGuard Home service as DNS on the G3, set
`ADGUARD_DNS_PORT=53`, complete the setup wizard at `http://<NAS-IP>:3080/`, then
set the G3's DNS server to `<NAS-IP>`. See the root
[README](../README.md#adguard-home-dns-on-the-g3).

## Updating cryanc

1. Edit `cryanc/Dockerfile` in this repo and push to `main`.
2. GitHub Actions rebuilds and publishes `ghcr.io/tommiec/cryanc-carl:latest`.
3. Pull and restart: `docker compose pull cryanc && docker compose up -d cryanc`.

## Updating Browservice

Browservice fetches the latest AppImage at Docker build time via the GitHub API.
The Dockerfile runs the AppImage extracted, sets the Chromium SUID sandbox helper
permissions, and disables GPU/Vulkan/background Chromium services for Docker.

1. Edit `browservice/Dockerfile` (bump a comment or the base image) and push to `main`.
2. GitHub Actions rebuilds and publishes `ghcr.io/tommiec/browservice:latest`.
3. Pull and restart: `docker compose pull browservice && docker compose up -d browservice`.

Do not push local builds to GHCR. Let GitHub Actions publish both images.
