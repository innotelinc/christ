# Merged image — all five services in one container

`compose.yml` runs the voice system as five containers: the front door, the console,
telno, the Cantus switch and the gateway. This directory builds the same five
services into **one image**, supervised as five processes, for a box that would
rather carry one container than five.

It builds from source in the estate tree, not from the published images: each
service's directory is copied out of the tree and only telno is compiled.

```sh
# from the estate root (the parent of christ/)
docker build -f christ/deploy/merged/Dockerfile -t christ-merged .
```

Run it with the same data the five containers used:

```sh
docker run -d --name christ-merged --restart unless-stopped \
  --env-file christ/.env \
  -p 8001:8001 -p 8002:8002 -p 8090:8090 -p 8188:8088 -p 7454:7454 \
  -p 5062:5060/udp -p 7443:7443/udp -p 7444:7444/udp -p 41000-41063:41000-41063/udp \
  -v /opt/christ/data:/data \
  -v /opt/christ/switch:/etc/cantus \
  -v /opt/christ/state/estate:/estate \
  -v /opt/christ/state/gateway:/data/gateway \
  christ-merged
```

| port | service | note |
|---|---|---|
| 8001 | christ | the front door |
| 8002 | console | moved off 8080 by `PORT` |
| 8090 | telno | moved off 8080 by `PORT` |
| 8088 → 8188 | switch | control + ARI |
| 7454 | gateway | health and control |
| 7443, 7444 (udp) | switch | Cantus and relay |
| 5062 → 5060 (udp) | gateway | SIP at the border; the fax listener moved to 5062 inside |
| 41000-41063 (udp) | gateway | media |

Five things the merge has to respect, all of them in the Dockerfile:

- **The front door and the gateway import the switch's modules.** `/jesus/apps/cantus-switch/src/router.js`
  and `/srv/cantus-switch` must exist as siblings; they are not optional copies.
- **Telno's entrypoint changes into `/app`** and generates its persisted secrets
  there, so telno keeps that path while the console lives under `/srv`.
- **The console and telno both default to 8080.** They are separated by `PORT`
  (8002 and 8090), not by patching either one.
- **compose renamed the carrier settings.** The gateway reads `SIP_*`, while the
  stack's `.env` keeps them as `VOIPMS_*` and lets compose do the renaming. It also
  pins `SIP_LOCAL_PORT=5060`; the code default is 0, an ephemeral port nothing can
  reach. `gateway-entrypoint` performs both, because one container has no compose.
- **The border carries a registrar expiry cap that only existed as a bind mount.**
  `gateway-sip.js` here is the live `apps/cantus-gateway/src/sip.js` plus
  `SIP_REGISTER_MAX_SECONDS`; without it a restart leaves every handset unregistered
  for up to the hour it negotiated. Upstreaming those lines retires the copy.

`CANTUS_CONTROL_TOKEN` and the rest of the stack's settings must be passed in
(`--env-file christ/.env`); without it telno boots but the switch refuses its
requests. Inside the container the services find each other on `127.0.0.1`: the
console's links to the switch, telno and the gateway point at their real ports
(`8088`, `8090`, `7454`), not at compose service names that do not resolve here.

This image deliberately shares one process tree and one network namespace. A crash
of one process is restarted by supervisor, but a container-level failure takes all
five down at once, and the UDP port ranges have to be published as a block. If that
is not what you want, use `compose.yml` instead — the five-container layout remains
the default.
