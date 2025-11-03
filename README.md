# docker-vpn-bridge

Give Docker containers access to one internal VPN‑only service **without using host‑network mode**.

That’s it. No tricks. No weird networking flags. Just a tiny bridge so your containers can talk to something inside a
VPN.

---

## Why this exists

By default, Docker containers cannot access a VPN interface that only the host is connected to.

The simple solution to run containers on the host network with `docker run --net host`:

- has to be applied to every single container
- is sometimes not feasible (container started from an IDE plugin)

---

## What you get

- ✅ Containers can reach one VPN‑internal endpoint
- ✅ No `--network host` for your app containers
- ✅ Does not expose VPN to outside host

What this is **not**:

- ❌ A full VPN gateway
- ❌ A VPN client
- ❌ General VPN routing for everything

Just a **port bridge**.

---

> These examples demonstrate how to reach http://10.7.2.100:8080 and the endpoint `/v1/models` on it.  
> http://10.7.2.100:8080 is an LLM server in a VPN and shows its models with `/v1/models`.  
> You can adapt this to your needs!

---

## Linux solutions

These solutions work on Linux.

### Linux Docker solutions

Principle: run one lightweight forwarder on the host (inside one container) and expose only one TCP port to Docker.

#### Architecture in one picture

```
Container ▶ http://172.17.0.1:18080  ──► Host VPN ──► 10.7.2.100:8080
```

Containers use the Docker bridge IP `172.17.0.1`.  
The host forwards traffic into the VPN.

#### Docker run

Set target IP/port and the local bridge port:

```bash
export TARGET_IP=10.7.2.100
export TARGET_PORT=8080

export LOCAL_PORT=18080
```

Set Docker bridge IP (docker0)

```bash
export DOCKER_BRIDGE_IP=$(ip -4 addr show docker0 | awk '/inet /{print $2}' | cut -d/ -f1)
```

Start the container

```bash
docker run -d \
  --name vpn-bridge \
  --network host \
  --restart unless-stopped \
  -p 127.0.0.1:${LOCAL_PORT}:${LOCAL_PORT}/tcp \
  alpine/socat \
  TCP-LISTEN:${LOCAL_PORT},bind=${DOCKER_BRIDGE_IP},reuseaddr,fork \
  TCP:${TARGET_IP}:${TARGET_PORT}
```

> Run the commands in [Testing](#testing) to see if it works.

Teardown:

```bash
docker rm -f vpn-bridge
```

#### docker-compose.yml

##### Docker bridge IP (docker0)

This uses the Docker bridge IP (docker0) which is host‑internal.  
Normally this is 172.17.0.1, read it with:

```bash
ip -4 addr show docker0 | awk '/inet /{print $2}' | cut -d/ -f1
```

##### .env file

Copy the [.env](.env) file and replace the values as needed.

##### docker-compose.yml file

Copy the [docker-compose.yml](docker-compose.yml) file to the same location.

##### Run

```bash
docker compose up -d
```

> Run the commands in [Testing](#testing) to see if it works.

Teardown:

```bash
docker compose down
```

---

## Windows notes (untested)

Windows approach (untested):

```
netsh interface portproxy add v4tov4 listenport=18080 listenaddress=127.0.0.1 connectport=8080 connectaddress=10.7.2.100
```

Then containers talk to:

```
http://host.docker.internal:18080
```

Docker‑inside‑Windows forwarding works only if the VPN allows NAT through the Docker VM. Many corporate VPNs don’t. Use
`netsh` instead.

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

Done. No `--network host` needed for your real containers.

---

## Troubleshooting

| Problem                                          | Meaning                                              |
|--------------------------------------------------|------------------------------------------------------|
 `curl localhost:18080` fails but bridge IP works | You bound only to Docker bridge. That’s intentional. |
 Windows container can't reach VPN IP             | Use `netsh portproxy`                                |

---

## Status matrix

| Platform | Method                 | Result     |
|----------|------------------------|------------|
 Linux    | Docker host‑net bridge | ✅ Works    |
 Linux    | systemd + socat        | ❓ Untested |
 Windows  | netsh forward          | ❓ Untested |
 Windows  | Docker forwarder       | ❓ Untested |