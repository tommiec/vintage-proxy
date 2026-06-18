# Proxy Stack for the iMac G3

Three proxies in one Container Manager project (`vintage-proxy`) for light web on the
iMac G3.

- **Macproxy** — primary choice on Tiger/Aquafox; fast, plain HTML, upstream certificate
  validation.
- **WebOne** — fallback when you want more layout/images; richer but slower.
- **Crypto Ancienne (`carl`)** — only for OS 9/Classilla special cases; no certificate
  validation.

No sensitive logins or banking through this machine. Macproxy/WebOne validate upstream
certificates, but between the iMac and the NAS the traffic is local proxy traffic. `carl`
is opportunistic compatibility, not a secure TLS solution.

A later VPN layer (gluetun → ProtonVPN) is deliberately not yet included; see the
[Future: VPN Layer](#future-vpn-layer) section at the bottom.

## Choice

Finding on the G3: pages load faster with Macproxy than with WebOne. Hence this order.

| Use | Proxy | Port | Note |
| --- | --- | --- | --- |
| Daily on Tiger/Aquafox | Macproxy | `5003` | Fastest, plain HTML. |
| More layout/images | WebOne | `8091` | Richer, heavier. |
| OS 9/Classilla | `carl` | `8767` | No certificate validation; not for Tiger. |

## Ports

| Service | Current host port | Internal |
|--------|-----------------------|--------|
| Macproxy | `5003` | `5001` |
| WebOne   | `8091` | `8080` |
| carl     | `8767` | `8765` |

Host ports are configurable via `.env`.

## Usage

1. Copy `.env.example` to `.env` and fill in at least `WEBONE_HOSTNAME`
   (the LAN IP or hostname of your Synology).
2. In Container Manager, create a new project pointing to this folder.
   carl is built locally from source, so the `cryanc/` folder with the `Dockerfile`
   must be present.
3. Build & start the project.

## Browser Configuration on the iMac

Use the proxy as the browser's proxy setting. Do not surf to the proxy as if it were a
website. WebOne may then show a "looped connection" page; that only means WebOne is
reachable.

In this setup, `NAS-IP` is currently `192.168.30.2`.

### Aquafox on Tiger

1. Open Aquafox → Preferences → Advanced → Network → Settings.
2. Choose **Manual proxy configuration**.
3. Set **HTTP Proxy** to `192.168.30.2`, port `5003`.
4. Set **SSL Proxy** also to `192.168.30.2`, port `5003`, or use the option
   "Use this proxy server for all protocols" if it is visible in your build.
5. Under **No Proxy for**, set at least `localhost, 127.0.0.1, 192.168.30.2`.
6. Test with `http://frogfind.com/` or `https://example.com/`.

Use WebOne as a fallback by temporarily setting the same fields to port `8091`. Do not
use `carl` in Aquafox.

### Safari / System Proxy on Tiger

Safari uses the OS X network settings:

1. System Preferences → Network → active interface (Ethernet) → **Proxies** tab.
2. Check **Web Proxy (HTTP)** and set server `192.168.30.2`, port `5003` for Macproxy,
   or `8091` for WebOne.
3. Check **Secure Web Proxy (HTTPS)** too with the same server and port.
4. Under bypass/exceptions, set at least `192.168.30.2`.
5. Save/apply and surf to a normal site, not to the proxy itself.

For Safari use the same port choice: Macproxy `5003`, WebOne `8091`. Do not use `carl`
for Safari/Aquafox.

## Test Matrix

Use this matrix only after Ethernet, DHCP, gateway, and DNS on the iMac G3 are
confirmed. For each test, note briefly: `works`, `partial`, `fails`, plus what you saw
(load time, layout, images, forms, error message).

Network basics: iMac `192.168.40.90`, NAS/proxy `192.168.30.2`. Cross-VLAN traffic works
with UDM rules for outbound and return traffic; see `docs/05-networking-and-wifi.md`.

| Test | Aquafox via Macproxy | Aquafox via WebOne | Safari via Macproxy | Safari via WebOne | OS 9/Classilla via carl | Notes |
|------|----------------------|--------------------|---------------------|-------------------|-------------------------|----------|
| Proxy reachable (direct `http://192.168.30.2:8091/`) | n/a | | n/a | | n/a | "Looped connection" = WebOne is reachable. |
| `http://example.com/` | | | | | | Plain HTTP basic test. |
| `https://example.com/` | | | | | | HTTPS/TLS basic test via proxy. |
| `https://frogfind.com/` | | | | | | Retro-friendly search/start page. |
| Wikipedia article | | | | | | Check text, links, and images. |
| Macintosh Garden download page | | | | | | Practical retro software source; no login needed. |
| Internet Archive item page | | | | | | Heavy page; expect possibly partial. |
| Modern JS-heavy site | | | | | | Negative control: note where it breaks. |
| Download a file via browser | | | | | | Test a small download, no large images. |

Recommended first round: test Aquafox via Macproxy, then the same sites via WebOne as a
fallback. Test Safari only as a system-proxy check. Test `carl` only when OS 9/Classilla
or another browser without `CONNECT` is concretely up next.

## Alternatives and Candidates for Later

Not part of the current stack, but documented for later evaluation. None of these is set
up here yet; the three-proxy stack above stays primary until one is actually tested on
the G3.

- **Browservice** (<https://github.com/ttalvitie/browservice>) — for modern sites that
  really need to work interactively. A modern browser runs on a server; the iMac mostly
  sees a rendering of it streamed as images. Practical, but less authentic than
  Macproxy/WebOne, and heavier on the network.
- **retro-proxy** (<https://github.com/DrKylstein/retro-proxy>) — an HTTPS→HTTP
  transcoding proxy in Node.js, with conversion/compression options and an `allowed.txt`
  to skip transcoding per site. Functionally in the same category as Macproxy/WebOne, but
  a Node/Yarn build rather than a ready-made Docker image, so it would need its own
  container and a G3 test before adoption. BSD-3-Clause.
- **ProtoWeb** (<https://protoweb.org>) — a hosted proxy service for browsing the *old*
  web: thousands of meticulously restored historical sites (and FTP/downloads), plus
  Wayback Machine access via a `URL:year` syntax. Different goal from the others: it does
  not transcode the live modern web but recreates the late-90s/early-2000s internet.
  Set it as the browser's HTTP proxy (server list at <https://protoweb.org/wiki/servers/>);
  free, no account needed, nothing to self-host. A good period-accurate complement to the
  live-web proxies above.

## Notes

- Use `carl` only for OS 9/Classilla special cases.
- Do not bind these proxies unprotected to the internet; they have no authentication.
  Keep them on your LAN.
- WebOne and Macproxy pull `:latest` and may be updated via Watchtower.
- `carl` image is published to GHCR by GitHub Actions; Watchtower updates it automatically.

## Updates

WebOne and Macproxy pull `:latest` and are updated automatically by Watchtower.

`carl` is published to `ghcr.io/tommiec/cryanc-carl:latest` by GitHub Actions whenever
`cryanc/**` changes on `main`. Watchtower picks up new image versions automatically.

To update `carl`:

1. Edit `cryanc/Dockerfile` (or bump `CRYANC_REF` to pin a specific upstream commit/tag).
2. Push to `main` — GitHub Actions rebuilds and publishes the image.
3. Watchtower picks it up on the next run, or force a redeploy in Portainer.
4. Test at least the proxy start page, `https://example.com/`, and the OS 9/Classilla
   case `carl` is meant for. Confirm working only after a short G3 test.

**Local builds** (Container Manager / `docker compose up --build`): `carl` is built
from source locally. Watchtower cannot refresh a local build; rebuild explicitly when
needed.

The `cryanc` container runs as a non-root user, with `no-new-privileges`, without extra
Linux capabilities, and with a simple TCP healthcheck on the internal `CARL_PORT`.

## Future: VPN Layer

The stack can later run through gluetun/ProtonVPN. That mainly helps with privacy, not
with browser security. It does not fix `carl`'s missing certificate validation.
