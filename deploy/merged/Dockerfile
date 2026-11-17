# syntax=docker/dockerfile:1.7
#
# One image for the five services compose.yml runs as five containers.
#
# Built from source, not from the service images: every stage below copies a
# directory out of the estate tree and, where a service has dependencies, installs
# them. The estate root is the build context, exactly as compose.yml builds the
# front door (`context: ..`, `dockerfile: christ/Dockerfile`):
#
#   docker build -f christ/deploy/merged/Dockerfile -t christ-merged .
#
# Only telno needs a build: the switch, the gateway, the console and the front door
# are dependency-free Node trees, which is why they are copied rather than compiled.

# ---------------------------------------------------------------- telno -----
FROM node:24-bookworm-slim AS telno-build
WORKDIR /build
# The bundle peaks well under this; an unbounded V8 heap starves a small box mid-build.
ENV NODE_OPTIONS=--max-old-space-size=640
COPY apps/telno/package.json apps/telno/package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY apps/telno/ ./
RUN npm run build

# -------------------------------------------------------------- runtime -----
FROM node:24-bookworm-slim

# supervisor is what makes one container hold five processes; ghostscript renders a
# fax document to the TIFF telno hands a T.38 gateway, and curl backs the healthchecks.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      supervisor curl ca-certificates tzdata ghostscript \
 && rm -rf /var/lib/apt/lists/*

# Telno's entrypoint changes into /app before it boots and generates the persisted
# secrets there, so this service keeps that path while the others live under /srv.
COPY --from=telno-build /build /app
WORKDIR /app
RUN npm prune --omit=dev
COPY apps/telno/docker/entrypoint.sh /usr/local/bin/telno-entrypoint

# The front door imports the switch's router rather than keeping a second copy of
# the dial-plan schema, so that path has to exist beside it.
COPY christ /jesus/christ
COPY apps/cantus-switch/src/router.js /jesus/apps/cantus-switch/src/router.js

# The switch, and the copy the gateway imports its protocol modules from.
COPY apps/cantus-switch /srv/switch
COPY apps/cantus-switch /srv/cantus-switch
COPY apps/cantus-gateway /srv/gateway
COPY estate/console /srv/console

COPY christ/deploy/merged/supervisord.conf /etc/supervisor/supervisord.conf

RUN mkdir -p /data/telno /data/switch /data/gateway /estate /etc/cantus \
 && chmod +x /usr/local/bin/telno-entrypoint

ENV NODE_ENV=production \
    ESTATE_DIR=/estate \
    DATA_DIR=/data/telno

# front door, console, telno, switch control, gateway health, and the voice ports
EXPOSE 8001 8002 8090 8088 7454 5060/udp 7443/udp 7444/udp

CMD ["supervisord", "-c", "/etc/supervisor/supervisord.conf", "-n"]
