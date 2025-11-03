# mitmproxy as forwarder (replaces socat)

## What this does

- mitmproxy listens on `${DOCKER_BRIDGE_IP}:${LOCAL_PORT}` and forwards to `${TARGET_IP}:${TARGET_PORT}`.
- mitm web UI is on `http://127.0.0.1:${MITM_WEB_PORT}` (host-only).
- No ports exposed off-host (proxy listener binds to the host's docker-bridge IP).

## When this works

- Target service is HTTP (or HTTP-like). mitmproxy is an HTTP proxy/reverse proxy.
- Clients connect over HTTP to the proxy endpoint (containers use `http://${DOCKER_BRIDGE_IP}:${LOCAL_PORT}`; host can
  also use that address).

## Limitations / caveats

- mitmproxy is NOT a generic raw TCP forwarder. If the target speaks non-HTTP protocols, stick with socat.
- For HTTPS interception, clients must trust mitmproxy CA or you must use TLS passthrough (`--mode reverse:https://...`)
  without interception.
- `network_mode: host` and binding to `${DOCKER_BRIDGE_IP}` assumes Linux; Docker Desktop behaves differently on
  macOS/Windows.

## Usage

### .env file

Copy the [.env](.env) file and replace the values as needed.

##### docker-compose.yml file

Copy the [docker-compose.yml](docker-compose.yml) file to the same location.

### Run

```bash
docker compose -f docker-compose-proxied.yml up -d
```

> Run the commands in [Testing](#testing) to see if it works.

### Teardown:

```bash
docker compose down
```

---

## Testing

### Test from host

```bash
curl http://172.17.0.1:18080/v1/models
```

### Test from another container

```bash
docker run --rm curlimages/curl:latest \
   -sS -m 5 http://172.17.0.1:18080/v1/models
```

### Mitm UI

```bash
open http://127.0.0.1:18083
```