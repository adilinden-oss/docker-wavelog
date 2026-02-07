ARG CADDY_VERSION=latest
ARG BUILD_VERSION=${CADDY_VERSION}-builder

FROM caddy:${BUILD_VERSION#latest-} AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:${CADDY_VERSION}

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
