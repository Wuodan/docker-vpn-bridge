# vpn-bridge + mitmproxy (mitm → bridge → target)

**Purpose:** Forward traffic to a VPN-only target via `socat`, with an optional logging path through `mitmproxy`. Nothing from mitm is reachable off-host.

## Variables

See the comments in the [.env](.env) file.

## How it flows

### Direct path (no logging)
```
host or container
  |
  |  (host: http://172.17.0.1:18080)
  |  (ctr : http://172.17.0.1:18080)
  v
[socat vpn-bridge @ 172.17.0.1:18080]
  |
  v
[target 10.7.2.100:8080]
```

### Logged path (mitm → bridge → target)
```
host or container
  |
  |  (host: http://172.17.0.1:18081)
  |  (ctr : http://172.17.0.1:18081)
  v
[socat vpn-bridge-logged @ 172.17.0.1:18081]
  |
  v
[mitmproxy listen 127.0.0.1:8080] --reverse--> [vpn-bridge 172.17.0.1:18080] --> [target 10.7.2.100:8080]

mitm UI (host only): http://127.0.0.1:8081  (password required)
```

## Quick tests
```
# DIRECT (host):
curl -sS http://172.17.0.1:18080

# LOGGED (host, goes via mitm):
curl -sS http://172.17.0.1:18081

# DIRECT (from a container):
docker run --rm curlimages/curl:8.11.1 -sS http://172.17.0.1:18080

# LOGGED (from a container):
docker run --rm curlimages/curl:8.11.1 -sS http://172.17.0.1:18081

# mitm UI (host only):
# open in browser: http://127.0.0.1:8081
```

## Short technical notes
- `network_mode: host` gives containers access to the host’s VPN routing; nothing is exposed off-host because mitm binds to `127.0.0.1`.
- `vpn-bridge` binds on `DOCKER_BRIDGE_IP` so **containers** can reach it; the host can also use that address.
- Logged path uses `vpn-bridge-logged` → `mitmproxy(127.0.0.1:8080)` → `vpn-bridge(172.17.0.1:18080)` → target.
- Avoid loops: mitm’s `--mode reverse:` points to the **direct** bridge (`172.17.0.1:18080`), never to its own listen port.
- On macOS/Windows Docker Desktop, `network_mode: host` differs or is unsupported; this layout targets Linux.
