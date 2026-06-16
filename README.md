# Sentinel — self-hosted RustDesk relay server

A minimal Docker Compose stack that runs your own **RustDesk** server so remote
sessions go through infrastructure you control instead of RustDesk's public
servers. It runs two services:

- **hbbs** — ID / rendezvous server: peer registration + NAT hole punching.
- **hbbr** — relay server: carries traffic when a direct P2P link can't be made.

```
   ┌────────────┐        ┌────────────┐
   │   hbbs     │◄──────►│   hbbr     │
   │ ID server  │        │   relay    │
   └─────▲──────┘        └─────▲──────┘
         │                     │
         └──────── your RustDesk clients ────────┘
              (controller + the machine being controlled)
```

> This repo is the **server only**. Install the normal RustDesk client app on
> each machine you want to connect to/from and point it at this server (see
> [DEPLOY.md](DEPLOY.md)).

## Quick start (local / LAN)

```bash
cp .env.example .env          # set RUSTDESK_SERVER_HOST to this host's IP
docker compose up -d          # starts hbbs + hbbr, generates the key pair
cat server/data/id_ed25519.pub  # the public key your clients need
```

Then in each RustDesk client's **Network settings**, set the *ID/Relay server*
to `RUSTDESK_SERVER_HOST` and paste that **Key**.

For deploying on a remote server (firewall ports, hardening, systemd-style
auto-start, client config), see **[DEPLOY.md](DEPLOY.md)**.

## Layout

```
docker-compose.yml   hbbs + hbbr service definitions
.env.example         configuration template (copy to .env)
.gitignore           keeps secrets + runtime data out of git
DEPLOY.md            remote-server deployment guide
server/data/         hbbs/hbbr keys + peer SQLite DB (generated on first boot)
```

## Ports

| Port        | Proto   | Service | Purpose                         |
|-------------|---------|---------|---------------------------------|
| 21115       | tcp     | hbbs    | NAT type test                   |
| 21116       | tcp+udp | hbbs    | ID registration / hole punch    |
| 21117       | tcp     | hbbr    | Relay                           |
| 21118       | tcp     | hbbs    | Web client (optional)           |
| 21119       | tcp     | hbbr    | Web client relay (optional)     |

## Notes

- `server/data/id_ed25519` is the server's **private key** — it is gitignored.
  Keep a secure backup; clients trust the matching public key.
- `RUSTDESK_SERVER_HOST` must be the address clients can actually reach (public
  IP or DNS), because hbbs hands it to clients as the relay endpoint.
