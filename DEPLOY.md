# Deploying the RustDesk relay server on a remote host

This guide covers running the `hbbs` + `hbbr` stack on a remote Linux server
(VPS, dedicated box, cloud VM) and pointing your RustDesk clients at it.

---

## 1. Prerequisites

- A Linux server with a **public IP** (or a DNS name pointing at it).
- **Docker** + the **Docker Compose plugin** installed:
  ```bash
  curl -fsSL https://get.docker.com | sh
  docker compose version   # confirm the plugin is available
  ```
- Ability to open the RustDesk ports in your firewall / cloud security group.

---

## 2. Get the code onto the server

```bash
git clone <your-repo-url> sentinel
cd sentinel
```

> `.env` and `server/data/` are gitignored, so a clone never carries secrets or
> another host's keys — each deployment generates its own.

---

## 3. Configure

```bash
cp .env.example .env
```

Edit `.env` and set `RUSTDESK_SERVER_HOST` to the **public IP or DNS name** that
clients will use to reach this server:

```ini
RUSTDESK_SERVER_HOST=relay.example.com
```

This value is critical: hbbs gives it to clients as the relay address, so it
must be reachable from the public internet — not an internal/Docker IP.

---

## 4. Open the firewall ports

The server needs these inbound ports (open them in your cloud security group
**and** the host firewall):

| Port  | Proto   | Purpose                      |
|-------|---------|------------------------------|
| 21115 | tcp     | NAT type test                |
| 21116 | tcp+udp | ID registration / hole punch |
| 21117 | tcp     | Relay                        |
| 21118 | tcp     | Web client (optional)        |
| 21119 | tcp     | Web client relay (optional)  |

> Note **21116 needs both TCP and UDP**. Missing the UDP rule is the most common
> cause of clients failing to register.

Examples:

```bash
# ufw (Debian/Ubuntu)
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp

# firewalld (RHEL/Fedora)
sudo firewall-cmd --permanent --add-port=21115-21119/tcp
sudo firewall-cmd --permanent --add-port=21116/udp
sudo firewall-cmd --reload
```

If you don't need the web client, you can omit 21118 and 21119.

---

## 5. Start the server

```bash
docker compose up -d
```

On first boot, hbbs generates the key pair under `server/data/`. Grab the public
key your clients will need:

```bash
cat server/data/id_ed25519.pub
```

Check both containers are healthy:

```bash
docker compose ps
docker compose logs -f          # Ctrl-C to stop following
```

`restart: unless-stopped` is set in the compose file, so the containers come
back automatically after a reboot or crash (as long as the Docker daemon is
enabled: `sudo systemctl enable --now docker`).

---

## 6. Point your RustDesk clients at it

On **each** machine (both the one you control from and the one being
controlled), open the RustDesk client → **Settings → Network → ID/Relay
server**, and set:

- **ID server:** `RUSTDESK_SERVER_HOST` (e.g. `relay.example.com`)
- **Relay server:** leave blank (hbbs supplies it) or the same host
- **Key:** the contents of `server/data/id_ed25519.pub`

Once configured, connect using the target machine's RustDesk ID + its password
as usual. Traffic now flows through your server.

---

## 7. Updating

```bash
cd sentinel
git pull                     # if config changed
docker compose pull          # fetch the latest rustdesk-server image
docker compose up -d         # recreate with the new image
```

Your keys and peer database in `server/data/` are preserved across updates.

---

## 8. Backups

The only stateful data is `server/data/`:

- `id_ed25519` / `id_ed25519.pub` — the server key pair (clients trust this).
- `db_v2.sqlite3*` — registered peers.

Back it up so you don't have to re-key every client after a host rebuild:

```bash
tar czf sentinel-data-$(date +%F).tgz server/data
```

> Treat `id_ed25519` as a secret. Anyone with it can impersonate your server.

---

## Troubleshooting

| Symptom                              | Likely cause / fix                                   |
|--------------------------------------|------------------------------------------------------|
| Clients can't register               | 21116 **UDP** not open, or wrong `RUSTDESK_SERVER_HOST` |
| Sessions connect then drop / no relay| 21117/tcp blocked                                    |
| "Key mismatch" in client             | Wrong key pasted; re-copy `server/data/id_ed25519.pub` |
| Containers not auto-starting on boot  | `sudo systemctl enable --now docker`                 |

View logs per service:

```bash
docker compose logs hbbs
docker compose logs hbbr
```
