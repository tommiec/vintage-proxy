# Crypto Ancienne (carl) - TLS-proxy voor geriatrische browsers.
#
# carl heeft geen officiële image en bindt zelf geen netwerkpoort: hij leest een
# volledige proxy-aanvraag van stdin en schrijft het antwoord naar stdout. We
# compileren hem uit de bron en serveren hem per connectie via socat, dat zich
# als een inetd-achtige superserver gedraagt.

# --- build: carl compileren ---
FROM alpine:3.20 AS build
ARG CRYANC_REF=main
RUN apk add --no-cache git gcc musl-dev
RUN git clone https://github.com/classilla/cryanc.git /src
WORKDIR /src
RUN git checkout "$CRYANC_REF"
RUN gcc -O3 -o carl carl.c

# --- runtime ---
FROM alpine:3.20
# netcat-openbsd: levert een nc met -z (BusyBox' nc kent die vlag niet).
RUN apk add --no-cache socat netcat-openbsd \
    && addgroup -S carl \
    && adduser -S -G carl -H -s /sbin/nologin carl
COPY --from=build /src/carl /usr/local/bin/carl

# CARL_PORT: interne luisterpoort. CARL_OPTS: opties voor carl(1).
ENV CARL_PORT=8765 \
    CARL_OPTS="-p -t"

EXPOSE 8765

USER carl

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD nc -z 127.0.0.1 "$CARL_PORT" || exit 1

# socat aanvaardt elke TCP-connectie en koppelt de socket aan carl's stdin/stdout.
CMD ["sh", "-c", "exec socat TCP-LISTEN:${CARL_PORT},reuseaddr,fork EXEC:\"/usr/local/bin/carl ${CARL_OPTS}\""]
