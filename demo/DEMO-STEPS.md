# WireGuard-in-container demo (host joins)

## What you get

- A container runs WireGuard (server) and a tiny HTTP server bound to the WireGuard IP.
- Your host joins the VPN and can reach `http://10.100.0.1:8080` only through the tunnel.

## Files

- `docker-compose.wg-demo.yml` — brings up the WireGuard endpoint + demo HTTP
- `wg-server/wg0.conf` — container WireGuard config (fill in keys)
- `wg-host.conf` — host WireGuard config (fill in keys)

## Steps (Linux host)

1) Install WireGuard tools on the host:

```bash
sudo apt-get update
sudo apt-get install -y wireguard
```

2) Generate keys (host + server), then fill placeholders in both configs:

```bash
SERVER_PRIVATE_KEY=$(wg genkey)
SERVER_PUBLIC_KEY=$(printf "%s" "$SERVER_PRIVATE_KEY" | wg pubkey)
HOST_PRIVATE_KEY=$(wg genkey)
HOST_PUBLIC_KEY=$(printf "%s" "$HOST_PRIVATE_KEY" | wg pubkey)

# Create local setup
mkdir -p local/wg-server/wg_confs
cp wg0.conf local/wg-server/wg_confs
cp wg-host.conf local/
# Copy and update files
sed \
  -e "s#__SERVER_PRIVATE_KEY__#${SERVER_PRIVATE_KEY}#" \
  -e "s#__HOST_PUBLIC_KEY__#${HOST_PUBLIC_KEY}#" \
  wg0.conf > local/wg-server/wg_confs/wg0.conf
sed \
  -e "s#__HOST_PRIVATE_KEY__#${HOST_PRIVATE_KEY}#" \
  -e "s#__SERVER_PUBLIC_KEY__#${SERVER_PUBLIC_KEY}#" \
  wg-host.conf > local/wg-host.conf
```

3) Start the container VPN and demo service:

```bash
docker compose -f docker-compose.wg-demo.yml up -d
```

4) Bring up the host side:

```bash
sudo wg-quick up ./local/wg-host.conf
```

5) Test from host with VPN:

```bash
curl -m 5 http://10.100.0.1:8080
```

> Expected: "Hello over WireGuard"

6) Test from other docker container without VPN

```bash
docker run --rm curlimages/curl:latest \
   -sS -m 5 http://10.100.0.1:8080
```

6) Tear down:

```bash
docker compose -f docker-compose.wg-demo.yml down
rm -rf local
sudo wg-quick down ./local/wg-host.conf
```

## Notes

- The demo confines traffic to the tunnel subnet (10.100.0.0/24). No default route changes.
- Because `demo` shares the `wg` network namespace, it is only reachable at `10.100.0.1:8080` via WireGuard.
- This is for testing connectivity and the concept of a container-hosted VPN only.
