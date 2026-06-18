# Proxy Stack for the iMac G3

Three-service Docker Compose stack providing legacy web access for the iMac G3.

- **Macproxy** — primary choice on Tiger/Aquafox; fast, plain HTML, upstream certificate
  validation.
- **WebOne** — fallback when you want more layout/images; richer but slower.
- **Crypto Ancienne (`carl`)** — only for OS 9/Classilla special cases; no certificate
  validation.

No sensitive logins or banking through this machine. Macproxy/WebOne validate upstream
certificates, but between the iMac and the NAS the traffic is local proxy traffic. `carl`
is opportunistic compatibility, not a secure TLS solution.

## Screenshots

![Aquafox on the iMac G3 loading the Wikipedia iMac G3 article via Macproxy](assets/screenshot.jpg)

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

1. Copy `.env.example` to `.env` and set at least `PROXY_HOSTNAME` to the IP
   or hostname of the machine running this stack.
2. Start the stack:
   ```sh
   docker compose up -d --build
   ```
   `carl` is compiled from source on first start; allow a few minutes for the build.
3. Configure the browser's proxy to the host IP and the relevant port (see [Ports](#ports)).

## Portainer CE

Portainer CE cannot use the local `build:` flow for `cryanc/carl`, so the Portainer
stack uses the GitHub Actions image instead:

```text
ghcr.io/tommiec/cryanc-carl:latest
```

See [deploy/](deploy/) for a pre-built-images stack and `.env` template. The real
deployment journal can still live in a separate GitOps repository.

## Browser Configuration on the iMac

Use the proxy as the browser's proxy setting. Do not surf to the proxy as if it were a
website. WebOne may then show a "looped connection" page; that only means WebOne is
reachable.

Replace `<NAS-IP>` in the steps below with the IP address or hostname of the machine
running this stack (e.g. `192.168.1.10`).

### Aquafox on Tiger

1. Open Aquafox → Preferences → Advanced → Network → Settings.
2. Choose **Manual proxy configuration**.
3. Set **HTTP Proxy** to `<NAS-IP>`, port `5003`.
4. Set **SSL Proxy** also to `<NAS-IP>`, port `5003`, or use the option
   "Use this proxy server for all protocols" if it is visible in your build.
5. Under **No Proxy for**, set at least `localhost, 127.0.0.1, <NAS-IP>`.
6. Test with `http://frogfind.com/` or `https://example.com/`.

Use WebOne as a fallback by temporarily setting the same fields to port `8091`. Do not
use `carl` in Aquafox.

### Safari / System Proxy on Tiger

Safari uses the OS X network settings:

1. System Preferences → Network → active interface (Ethernet) → **Proxies** tab.
2. Check **Web Proxy (HTTP)** and set server `<NAS-IP>`, port `5003` for Macproxy,
   or `8091` for WebOne.
3. Check **Secure Web Proxy (HTTPS)** too with the same server and port.
4. Under bypass/exceptions, set at least `<NAS-IP>`.
5. Save/apply and surf to a normal site, not to the proxy itself.

For Safari use the same port choice: Macproxy `5003`, WebOne `8091`. Do not use `carl`
for Safari/Aquafox.

## Test Matrix

Use this matrix only after Ethernet, DHCP, gateway, and DNS on the iMac G3 are
confirmed. For each test, note briefly: `works`, `partial`, `fails`, plus what you saw
(load time, layout, images, forms, error message).

Network basics: replace `<IMAC-IP>` and `<NAS-IP>` with your actual addresses.
If the iMac and the proxy host are on different VLANs, ensure your router allows
cross-VLAN traffic between them.

| Test | Aquafox via Macproxy | Aquafox via WebOne | Safari via Macproxy | Safari via WebOne | OS 9/Classilla via carl | Notes |
|------|----------------------|--------------------|---------------------|-------------------|-------------------------|----------|
| Proxy reachable (direct `http://<NAS-IP>:8091/`) | n/a | | n/a | | n/a | "Looped connection" = WebOne is reachable. |
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

Adjacent browsing extras:

- **AdGuard Home** DNS adblocking is a useful later browsing improvement for Aquafox/Tiger,
  but it belongs beside the NAS/browser route rather than inside this three-proxy stack.

## carl / cryanc — protocol notes

Carl is **not** a standard HTTP CONNECT proxy. It uses a socat/inetd model:
each incoming TCP connection forks a `carl` process that reads a raw HTTP
request from stdin and writes the response to stdout. Carl handles the TLS
negotiation to the upstream server on behalf of the client.

This means:

- **Classilla / OS 9**: sends a full `GET http://example.com/ HTTP/1.0` to the
  proxy — exactly what carl expects. TLS is transparent to the browser.
- **Modern browsers** (Safari, Firefox, curl with HTTPS): send `CONNECT
  example.com:443 HTTP/1.1` — carl does not understand CONNECT and exits with
  status 1. Socat logs `child exited with status 1`. **This is expected.**

### Verifying carl from a modern machine

Check the port is open:

```sh
nc -z <NAS-IP> 8767 && echo "open"
```

Test plain HTTP (not HTTPS) through carl:

```sh
curl --proxy http://<NAS-IP>:8767 http://neverssl.com/
```

HTTPS through carl cannot be tested from a modern client — that requires a
browser like Classilla that does not use CONNECT.

## Notes

- Use `carl` only for OS 9/Classilla; modern browsers must use Macproxy or WebOne.
- Do not bind these proxies unprotected to the internet; they have no authentication.
  Keep them on your LAN.
- WebOne and Macproxy can be updated with `docker compose pull && docker compose up -d`.
- `carl` is published to `ghcr.io/tommiec/cryanc-carl:latest` by GitHub Actions; pull to update.

## Updates

**WebOne and Macproxy:**

```sh
docker compose pull && docker compose up -d
```

**`carl`:**

`carl` has no official image. GitHub Actions builds and publishes
`ghcr.io/tommiec/cryanc-carl:latest` on every push to `main` that touches `cryanc/**`.

To update:

1. Edit `cryanc/Dockerfile` (or bump `CRYANC_REF` to pin a specific upstream commit/tag).
2. Push to `main` — GitHub Actions rebuilds and publishes the image.
3. Pull the new image and restart: `docker compose pull cryanc && docker compose up -d cryanc`.
4. Test at least `https://example.com/` and the OS 9/Classilla case `carl` is meant for.
   Confirm working only after a short G3 test, not just because the container starts.

The `cryanc` container runs as a non-root user, with `no-new-privileges`, without extra
Linux capabilities, and with a simple TCP healthcheck on the internal `CARL_PORT`.
