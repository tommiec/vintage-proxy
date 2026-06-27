# Browservice — server-side Chromium rendering for legacy browsers.
#
# Browservice runs a full Chromium instance on the server and streams the
# rendered page as JPEG frames to the client over HTTP. The iMac only needs
# to display images and send click/key events — no JavaScript, no TLS, no
# modern CSS at the client side.
#
# We use the official AppImage from ttalvitie/browservice (GitHub Releases)
# and extract it with --appimage-extract to avoid the FUSE requirement inside
# Docker. The GitHub API is queried at build time so every image build picks
# up the latest release automatically.
#
# The Chromium SUID sandbox helper (chrome-sandbox) must be root-owned with
# mode 4755 — see the RUN step below. seccomp=unconfined is set in compose.yaml
# because Docker's default profile blocks some syscalls Chromium needs.

FROM ubuntu:22.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
       jq \
    && rm -rf /var/lib/apt/lists/*

# Download the latest official x86_64 AppImage, extract in-place, discard archive.
RUN set -eux; \
    APPIMAGE_URL=$(curl -fsSL \
        https://api.github.com/repos/ttalvitie/browservice/releases/latest \
        | jq -r '[.assets[] | select(.name | test("x86_64.*\\.AppImage$"; "i"))] | .[0].browser_download_url'); \
    echo "Fetching: $APPIMAGE_URL"; \
    curl -fsSL "$APPIMAGE_URL" -o /opt/browservice.AppImage; \
    chmod +x /opt/browservice.AppImage; \
    cd /opt && ./browservice.AppImage --appimage-extract; \
    mv /opt/squashfs-root /opt/browservice; \
    rm /opt/browservice.AppImage

# The Chromium SUID sandbox helper must be owned by root with the setuid bit.
# This is done at build time (as root) before switching to the app user.
RUN chown root:root /opt/browservice/opt/browservice/chrome-sandbox \
    && chmod 4755 /opt/browservice/opt/browservice/chrome-sandbox

RUN useradd -r -s /sbin/nologin -m -d /home/browservice browservice \
    && mkdir -p /home/browservice/.browservice /tmp/.X11-unix \
    && chown -R browservice:browservice /home/browservice \
    && chown root:root /tmp/.X11-unix \
    && chmod 1777 /tmp/.X11-unix

EXPOSE 8080

USER browservice
ENV HOME=/home/browservice
WORKDIR /home/browservice

# --vice-opt-http-listen-addr is the current option name (--http-listen-addr is deprecated).
CMD ["/opt/browservice/AppRun", "--vice-opt-http-listen-addr=0.0.0.0:8080"]
