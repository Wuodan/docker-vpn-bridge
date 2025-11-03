# docker-vpn-bridge

Give Docker containers access to one internal VPN‑only service **without using host‑network mode**.

That’s it. No tricks. No weird networking flags. Just a tiny bridge so your containers can talk to something inside a
VPN.

---

## Why this exists

By default, docker containers cannot access a VPN the host is on.

The simple solution to run the container on the host network with `docker run --net host`:

- has to be applied to every single container
- is sometimes not feasible (container started from an IDE plugin)

---

## What you get

- ✅ Containers can reach one VPN‑internal endpoint
- ✅ No `--network host` for your app containers
- ✅ Transparent, tiny, easy to debug
- ✅ Does not expose VPN to outside host

What this is **not**:

- ❌ A full VPN gateway
- ❌ A VPN client
- ❌ General VPN routing for everything

Just a **port bridge**.

---

## Architecture in one picture

```
Container ▶ http://172.17.0.1:18080  ──► Host VPN ──► 10.7.2.100:8080
```

Containers use the Docker bridge IP `172.17.0.1`.  
The host forwards traffic into the VPN.

> These examples demonstrate how to reach http://10.7.2.100:8080 and the endpoint `/v1/models` on it.  
> http://10.7.2.100:8080 is an LLM server in a VPN and shows its models with `/v1/models`.  
> You can adapt this to your needs!

---

## Quick start (Linux + Docker)

Goals:

- containers reach `10.7.2.100:8080` on VPN
- VPN bridge is not available outside the host

### docker run

Set target ip and port plus local port:

```bash
export TARGET_IP=10.7.2.100
export TARGET_PORT=8080

export LOCAL_PORT=18080
```

Set **Docker bridge IP** (docker0)
```bash
export DOCKER_BRIDGE_IP=$(ip -4 addr show docker0 | awk '/inet /{print $2}' | cut -d/ -f1)
```

Start the container

```bash
docker run -d \
  --name vpn-bridge \
  --network host \
  -p 127.0.0.1:${LOCAL_PORT}:${LOCAL_PORT}/tcp \
  alpine/socat \
  TCP-LISTEN:${LOCAL_PORT},bind=${DOCKER_BRIDGE_IP},reuseaddr,fork \
  TCP:${TARGET_IP}:${TARGET_PORT}
```

> Run the commands in [Testing](#testing) to see if it works.

### docker-compose.yml

#### Docker bridge IP (docker0)

This uses the **Docker bridge IP** (docker0) which is host-internal.  
Normally this is 172.17.0.1, read it with:

```bash
ip -4 addr show docker0 | awk '/inet /{print $2}' | cut -d/ -f1
```

#### .env file

Copy the [.env](.env) file and replace the values as needed.

#### docker-compose.yml file

Copy the [docker-compose.yml](docker-compose.yml) file to the same location.

#### Run

```bash
docker compose up -d
```

### Testing

#### Test from host

```bash
curl http://172.17.0.1:18080/v1/models
```

#### Test from another container

```bash
docker run --rm curlimages/curl:latest \
   -sS -m 5 http://172.17.0.1:18080/v1/models
```

Done. No `--network host` needed for your real containers.

---

## Windows notes

Best option on Windows:

```
netsh interface portproxy add v4tov4 ...
```

Then containers talk to:

```
http://host.docker.internal:PORT
```

Docker‑inside‑Windows forwarding works only if the VPN allows NAT through the Docker VM. Many corporate VPNs don’t. Use
`netsh` instead.

---

## Troubleshooting

| Problem                                          | Meaning                                                     |
|--------------------------------------------------|-------------------------------------------------------------|
 `curl localhost:18080` fails but bridge IP works | You bound only to Docker bridge. That’s intentional.        |
 Windows container can't reach VPN IP             | Use `netsh portproxy`                                       |
 Container hits proxy but VPN service won't reply | Corporate VPN blocks container namespace traffic (expected) |

---

## Status matrix

| Platform | Method                 | Result                        |
|----------|------------------------|-------------------------------|
 Linux    | Docker host‑net bridge | ✅ Works                       |
 Linux    | systemd + socat        | ✅ Works                       |
 Windows  | netsh forward          | ✅ Works                       |
 Windows  | Docker forwarder       | ⚠️ Sometimes (depends on VPN) |

---

## License

MIT by default.  
Add “no AI training” clause if you want — your choice.

---

## TL;DR

Enable container access to one VPN‑internal service **without using host networking**.

---

