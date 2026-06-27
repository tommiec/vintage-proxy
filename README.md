# Proxy Stack for the iMac G3

Docker Compose stack providing legacy web access for the iMac G3.

- **[Macproxy](https://github.com/rdmark/macproxy)** — primary choice on Tiger/Aquafox; fast, plain HTML, upstream certificate
  validation.
- **[WebOne](https://github.com/u306060/webone)** — fallback when you want more layout/images; richer but slower.
- **[Crypto Ancienne (`carl`)](https://github.com/classilla/cryanc)** — only for OS 9/Classilla special cases; no certificate
  validation.
- **[Browservice](https://github.com/ttalvitie/browservice)** — server-side Chromium rendering; streams pages as JPEG frames so the
  iMac needs no JavaScript, TLS, or CSS support. Treat it as a candidate for modern
  sites that break with the other proxies; Tiger and OS 9/Classilla still need
  validation on the G3, and PowerPC/m68k support is not tested upstream.
- **[AdGuard Home](https://github.com/AdguardTeam/AdGuardHome)** — standalone DNS blocker; optional for the G3 and independent of the
  browser proxy choice.

No sensitive logins or banking through this machine. Macproxy/WebOne validate upstream
certificates, but between the iMac and the NAS the traffic is local proxy traffic. `carl`
is opportunistic compatibility, not a secure TLS solution.

## Screenshots

![Aquafox on the iMac G3 loading the Wikipedia iMac G3 article via Macproxy](assets/screenshot.jpg)

## Choice

Finding on the G3: pages load faster with Macproxy than with WebOne. Hence this order.

**Macproxy, WebOne, and carl are HTTP proxies** — you configure them once in the OS or browser
proxy settings and then browse normally with Safari's own address bar. The proxy works invisibly
in the background.

**Browservice is different** — do not configure it as a proxy. Instead, surf directly to
`http://<NAS-IP>:8083/` and type the destination URL inside the Browservice interface. You are
then looking at a live Chromium browser running on the server, streamed as images to Safari.
Use it as a last resort when Macproxy and WebOne cannot handle a site.

| Use | Proxy | Port | How to use |
| --- | --- | --- | --- |
| Daily on Tiger/Aquafox | Macproxy | `5003` | Set as HTTP proxy in OS/browser settings. |
| More layout/images | WebOne | `8091` | Set as HTTP proxy in OS/browser settings. |
| OS 9/Classilla | `carl` | `8767` | Set as HTTP proxy; no certificate validation; not for Tiger. |
| Modern sites (last resort) | Browservice | `8083` | Surf to `http://<NAS-IP>:8083/` and type URL inside. Not validated on the G3 yet. |

## Ports

| Service | Current host port | Internal |
|--------|-----------------------|--------|
| Macproxy | `5003` | `5001` |
| WebOne   | `8091` | `8080` |
| carl     | `8767` | `8765` |
| Browservice | `8083` | `8080` |
| AdGuard Home DNS   | `5354` (set `53` in `.env` for G3 use) | `53` |
| AdGuard Home setup | `3080` | `3000` |
| AdGuard Home admin | `3001` | `80` |

Host ports are configurable via `.env`.

## Usage

1. Copy `.env.example` to `.env` and set at least `PROXY_HOSTNAME` to the IP
   or hostname of the machine running this stack.
2. Start the stack:
   ```sh
   docker compose up -d --build
   ```
   `carl` and `browservice` are compiled from source on first start; allow a few minutes
   for the build.
3. Configure the browser's proxy to the host IP and the relevant port (see [Ports](#ports)).
   Browservice is not a proxy — open `http://<NAS-IP>:8083/` directly in the browser instead.
4. Optional: use [AdGuard Home DNS](#adguard-home-dns-on-the-g3) for G3-wide DNS blocking.

## Portainer CE

Portainer CE cannot use the local `build:` flow for `cryanc/carl` or `browservice`,
so the Portainer stack uses the GitHub Actions images instead:

```text
ghcr.io/tommiec/cryanc-carl:latest
ghcr.io/tommiec/browservice:latest
```

See [deploy/](deploy/) for a pre-built-images stack and `.env` template. The real
deployment journal can still live in a separate GitOps repository.

## Browser Configuration on the iMac

**Macproxy and WebOne** are configured once as the system HTTP proxy. After that, Safari and
Aquafox use their own address bar as normal — the proxy is invisible.

**Browservice** is not configured as a proxy. Open `http://<NAS-IP>:8083/` directly in Safari,
then use the address bar inside that page. Use it only when Macproxy/WebOne fail on a site.

For Macproxy and WebOne: do not surf to the proxy URL itself. WebOne will show a "looped
connection" page if you do — that just means it is reachable.

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

## Browservice

Browservice runs a full Chromium instance on the server and streams the rendered page
as JPEG frames over HTTP to the client browser. The iMac sends only mouse/keyboard
events and receives images — no JavaScript execution, no TLS handshake, no modern CSS
parsing happens on the G3 side.

Use it when Macproxy or WebOne cannot render a page you actually need. It is a
candidate for Tiger browsers (Aquafox, Safari) and OS 9/Classilla because the client
only needs to load images and submit basic form data, but it still needs validation
on the G3.

### Browser configuration for Browservice

Do **not** add Browservice to the proxy settings. Open `http://<NAS-IP>:8083/` directly
in Safari or Aquafox as a regular URL. Browservice presents its own start page with an
address bar; type the destination URL there. All rendering happens on the server — Safari
only receives images and sends back mouse/keyboard events.

### Updating Browservice

Browservice fetches the latest AppImage from the official GitHub Releases at Docker
build time. To update:

1. Edit `browservice/Dockerfile` (bump a comment or the base image) and push to `main`.
2. GitHub Actions rebuilds and publishes `ghcr.io/tommiec/browservice:latest`.
3. Watchtower pulls the new image and restarts the container.

### Docker notes

Browservice uses Chromium/CEF. The `chrome-sandbox` binary is set to root-owned mode
`4755` in the Dockerfile so the SUID sandbox can run as the non-root `browservice`
user. The compose file grants `SYS_ADMIN` and sets `seccomp=unconfined` because
Synology DSM disables unprivileged user namespaces and Docker's default seccomp
profile blocks some Chromium syscalls. Do not add `no-new-privileges:true` to
Browservice; it blocks the setuid bit. Browservice keeps its runtime home inside
the container; no host bind mount is needed unless we later decide to preserve
browser sessions, cache, or installed fonts across container replacement. Do not
run this proxy on an internet-exposed host without additional network-level
protection.

## AdGuard Home DNS on the G3

AdGuard Home is included as a standalone DNS service. It is not wired into the proxy
containers: proxy choice stays in the browser, DNS choice stays in OS X Network settings.

To use it from the G3:

1. Set `ADGUARD_DNS_PORT=53` in `.env` before starting the stack. Standard DNS clients
   use port `53`; the default `5354` is only a collision-safe startup value.
2. Start the stack and finish the setup wizard at `http://<NAS-IP>:3080/`.
3. In the wizard, keep DNS listening on internal port `53`. Use internal port `80` for
   the admin UI if you want to reach it at `http://<NAS-IP>:3001/` after setup.
4. On Tiger: System Preferences → Network → active interface → **DNS** tab. Add
   `<NAS-IP>` as the DNS server.

This filters DNS for all G3 traffic. It does not change which proxy the browser uses.

### Post-install: Retro Light filtering

Treat AdGuard Home as a light DNS filter for the G3, not as an aggressive adblock
setup. The goal is fewer tracker, ad, metrics, and pixel requests while keeping old
sites usable.

In the setup wizard:

- Keep DNS listening on internal port `53`.
- Keep the admin UI on internal port `80` so it remains reachable at
  `http://<NAS-IP>:3001/`.
- Use your normal LAN resolver or router as upstream DNS. For a NAS in a server VLAN
  behind a UDM/UniFi gateway, this is usually that VLAN gateway, for example
  `192.168.30.1`.
- Use plain DNS upstreams first. DoH/DoT is not needed for this use case.

In **Filters -> DNS blocklists**, start small:

```text
AdGuard DNS filter
https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt

HaGeZi Multi LIGHT
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/light.txt
```

If that is stable but still lets too much tracking through, replace HaGeZi LIGHT with:

```text
HaGeZi Multi NORMAL
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/normal.txt
```

Avoid stacking aggressive Pro, Ultimate, gambling, social, annoyance, or cookie lists
at first. They are more likely to break old browsing than to help the G3.

Recommended DNS settings:

- Protection: on.
- Blocking mode: `NXDOMAIN`.
- Blocked response TTL: `3600`.
- Cache size: `64 MB` or higher.
- Optimistic cache: on, if available.
- Minimum TTL: `300`.
- Maximum TTL: `86400`.
- Safe Search, parental control, and browsing security: off for the first test round.

Only add custom filtering rules when the query log shows they are still needed:

```text
||googletagmanager.com^
||google-analytics.com^
||doubleclick.net^
||googlesyndication.com^
||facebook.net^
||connect.facebook.net^
||scorecardresearch.com^
||hotjar.com^
||newrelic.com^
||sentry.io^
||segment.io^
```

Do not broadly block shared infrastructure such as `gstatic.com`, Cloudflare, Akamai,
Fastly, or common JavaScript CDNs. That usually breaks more pages than it speeds up.

Final check: open **Query Log** in AdGuard Home, filter on the G3 IP, and browse with
Aquafox/Macproxy. If you see little or no traffic from the G3, the browser path is
probably resolving DNS inside the proxy container instead. In that case AdGuard still
filters direct G3 traffic, but proxy-side DNS filtering needs a separate decision.

References:

- <https://github.com/AdguardTeam/AdGuardHome/wiki>
- <https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration>
- <https://github.com/hagezi/dns-blocklists>

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

### Browservice test

Open `http://<NAS-IP>:8083/` directly in Safari or Aquafox (no proxy setting needed).
Type a destination URL in the Browservice address bar and note the result.

| Test | Safari via Browservice | Aquafox via Browservice | Notes |
|------|------------------------|-------------------------|-------|
| Browservice start page loads | | | Basic reachability. |
| `https://example.com/` | | | Simple HTTPS page. |
| Modern JS-heavy site | | | Main use case for Browservice. |
| Video page (e.g. YouTube) | | | Expect audio-only or partial at best. |

## Alternatives and Candidates for Later

Not part of the current stack, but documented for later evaluation.

- **retro-proxy** (<https://github.com/DrKylstein/retro-proxy>) — an HTTPS→HTTP
  transcoding proxy in Node.js, with conversion/compression options and an `allowed.txt`
  to skip transcoding per site. Functionally in the same category as Macproxy/WebOne, but
  a Node/Yarn build rather than a ready-made Docker image, so it would need its own
  container and a G3 test before adoption. BSD-3-Clause.

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

## Credits

This repository is a Docker Compose wrapper. All the actual work is done by these upstream
projects — credits and thanks go to their authors and contributors:

| Project | Author | Repository |
|---------|--------|------------|
| Macproxy | rdmark | <https://github.com/rdmark/macproxy> |
| WebOne | u306060 | <https://github.com/u306060/webone> |
| Crypto Ancienne (carl) | Cameron Kaiser | <https://github.com/classilla/cryanc> |
| Browservice | Topi Talvitie | <https://github.com/ttalvitie/browservice> |
| AdGuard Home | AdGuard Team | <https://github.com/AdguardTeam/AdGuardHome> |
| HaGeZi DNS blocklists | hagezi | <https://github.com/hagezi/dns-blocklists> |
