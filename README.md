# docker-vpn-bridge

Give Docker containers access to one internal VPN‑only service **without using host‑network mode**.

That’s it. No tricks. No weird networking flags. Just a tiny bridge so your containers can talk to something inside a
corporate VPN.

---

## Why this exists

Some VPNs:

- run only on the host
- require browser login + MFA (SMS / Authenticator)
- break if you try to run them inside Docker
- block traffic unless it originates from the host network namespace

So your containers suddenly can’t reach `10.x.x.x` or `internal.company.local`.

Traditionally you'd use:

```
docker run --network host  # ugh
```

But that exposes everything and often breaks Compose setups.

This project gives you a **safer, minimal alternative**:
forward *one local port → one VPN IP:PORT*  
and let containers use that port normally.

---

## What you get

- ✅ Containers can reach one VPN‑internal endpoint
- ✅ No `--network host` for your app containers
- ✅ Works with browser‑based VPN logins
- ✅ Linux & Windows instructions
- ✅ Transparent, tiny, easy to debug

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

Containers use the Docker bridge IP.  
The host forwards traffic into the VPN.

---

## Quick start (Linux + Docker)

Goals:

- container reaches `10.7.2.100:8080`
- VPN bridge is not available outside the host

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

### Run

```bash
docker compose up -d
```

### Test from host

```bash
curl http://172.17.0.1:18080/
```

### Test from another container

```bash
docker run --rm curlimages/curl:latest \
   -sS -m 5 http://172.17.0.1:18080/
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

