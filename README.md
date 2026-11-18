[README.md](https://github.com/user-attachments/files/35054443/README.md)
# Deploying Artex on VirtualBox

A step-by-step build of the Artex autonomous pentest platform inside an Ubuntu Server VM — from an empty VirtualBox machine through Docker, the source build, the Claude model, and the first authorized task.

- **VirtualBox** 7.2
- **Guest OS** Ubuntu Server 26.04 LTS
- **Runtime** Docker Compose
- **Model** Claude Opus 5.5 (Anthropic)
- **Build time** ~60 min (first build)

> **Authorization first.** Artex runs real scanners against whatever you point it at. Only ever target systems you own or hold **written** permission to test. A job title or admin access is not authorization. Keep the first run on a local practice target.

Commands are labelled by where they run:

- **Host** — Windows PowerShell on your laptop
- **VM** — the SSH session inside the Ubuntu guest (every `docker compose` command runs from `~/ARTEX`)

---

## How the pieces connect

```
Windows laptop (host)
├─ Browser  ──tunnel──►  sshd (VM, port 22)
│  localhost:8787
│
└─ VirtualBox ─► Ubuntu VM (guest 10.0.2.15)
                 └─ Docker network: artex_default
                    ├─ artex-artex-1     Go server + web UI, 8 agents, scanners
                    │                     binds 127.0.0.1:8787-8788
                    │                     └──outbound──► Claude API (api.anthropic.com)
                    └─ artex-postgres-1   PostgreSQL 16, volume artex_pgdata, port 5432 (internal)
```

The web UI binds to `127.0.0.1` **inside the VM** only — it is never exposed on the network. The SSH tunnel bridges your laptop's browser to it.

---

## 00 — Prerequisites

On the Windows host, before touching the VM:

- **VirtualBox 7.2+** installed.
- **Ubuntu Server 26.04 LTS** ISO downloaded (the `live-server-amd64` image).
- **CPU virtualization** enabled in BIOS. On Windows, Hyper-V / Core Isolation can slow the VM — disable Memory Integrity if the guest clock drifts badly.
- **Hardware headroom:** 4 vCPUs, 8 GB RAM, ~40 GB free disk. The build pulls several GB of images.
- An **Anthropic Console account** with API credit (separate from a claude.ai subscription).

---

## 01 — Create and configure the VM

A NAT Network gives the VM internet access and lets you forward a port back to the host for SSH.

| Setting | Value |
|---|---|
| Type | Linux · Ubuntu (64-bit) |
| Memory | 8000 MB |
| CPUs | 4 |
| Disk | 40 GB+ (VDI, dynamically allocated) |
| Network | NAT Network (see below) |

Create a NAT Network with an SSH port-forward (host). This creates the network, enables DHCP, and forwards host `127.0.0.1:2222` to the VM's SSH port.

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
.\VBoxManage natnetwork add --netname ArtexNet --network "10.0.2.0/24" --enable --dhcp on
.\VBoxManage natnetwork modify --netname ArtexNet --port-forward-4 "ssh:tcp:[127.0.0.1]:2222:[10.0.2.15]:22"
```

Attach the VM's network adapter to **NAT Network → ArtexNet** in the VM's settings. The guest comes up as `10.0.2.15`.

---

## 02 — Install Ubuntu & enable SSH

Boot the VM from the ISO and run the installer:

- Choose **Ubuntu Server** (not minimized).
- Accept the guided full-disk layout.
- Create your user — this guide assumes `moran` with hostname `artex`.
- On the SSH screen, tick **Install OpenSSH server**.
- Reboot when prompted and remove the ISO from the virtual drive.

Connect from the host:

```powershell
ssh -p 2222 moran@127.0.0.1
```

Update the system (VM):

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 03 — Install Docker Engine

Install Docker and the Compose plugin from Docker's official script, then allow your user to run it without sudo (VM):

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# log out and back in so the group takes effect, then verify:
docker --version && docker compose version
```

> Re-connect your SSH session after `usermod` — the new `docker` group only applies to a fresh login.

---

## 04 — Get the Artex source & build

Place the project in `~/ARTEX`, then build and start the stack. The first build compiles the Go server, builds the web UI, and installs the scanner toolchain — expect roughly an hour.

```bash
cd ~/ARTEX
docker compose up -d --build
```

When it finishes you should see both containers created and started:

```
✓ Container artex-postgres-1  Healthy
✓ Container artex-artex-1     Started
```

Confirm it is running:

```bash
docker compose ps
```

The app binds to `127.0.0.1:8787-8788` inside the VM, and Postgres reports `healthy`.

---

## 05 — Open the web UI over an SSH tunnel

Because the UI only listens on the VM's localhost, forward the ports from the host. Leave this window open while you work (host):

```powershell
ssh -p 2222 -L 8787:127.0.0.1:8787 -L 8788:127.0.0.1:8788 moran@127.0.0.1
```

Then browse the host to <http://localhost:8787>. You'll land on the first-run setup page.

---

## 06 — Set the admin password

Artex prints a one-time setup token to its log on first start. Grab it (VM):

```bash
docker compose logs artex | grep -i -A2 token
```

Paste the token into **Setup token**, choose a password of at least 12 characters, and sign in.

> The token is **one-time** — after the password is set you log in with the password alone. If you lose it, `reset-password.sh` in `~/ARTEX` issues a fresh token.

---

## 07 — Connect Claude as the model

**System → LLM → new model profile.** Fill it in, save, then **Set as active** — profiles are not activated automatically.

| Field | Value |
|---|---|
| Name | Claude |
| Format | Anthropic |
| Model | `claude-opus-5-5` |
| Base URL | `https://api.anthropic.com` |
| API Key | your `sk-ant-…` key |
| Proxy / headers | leave blank |

Click **Test connection**. A sanity check from the VM confirms the container itself can reach the API (`401` means reachable, key aside):

```bash
docker exec artex-artex-1 curl -sS -o /dev/null -w '%{http_code}\n' https://api.anthropic.com/v1/models
```

> **"Credit balance too low"** means the key works but the account has no API credit. Add credit in the Anthropic Console → Plans & Billing, and set a monthly spend limit before large runs.

---

## 08 — Verify agents, tools & scanners

With the active profile saved, all agents use Claude automatically. Confirm the stack is whole:

- **System → Agent** — 8 agents (Goal Decomposer, Planner, Worker, Main, Auto, Penetration Testing, Report writer, Vulnerability Retest).
- **System → Tools** — 59 internal system tools, bound to their agents.

Check the external scanners are on the container's PATH (VM):

```bash
docker exec artex-artex-1 sh -c 'for t in nuclei httpx subfinder ffuf dalfox sqlmap bbot playwright-cli rg jq; do printf "%-15s" "$t"; command -v $t >/dev/null && echo OK || echo MISSING; done'
```

All lines should read `OK`.

---

## 09 — Run your first task (on a safe target)

Prove the platform end-to-end against a deliberately vulnerable app on the same Docker network. OWASP Juice Shop is built for this.

Launch the practice target (VM):

```bash
docker run -d --name juice-shop --network artex_default --restart unless-stopped bkimminich/juice-shop
# confirm Artex can reach it — expect 200
docker exec artex-artex-1 curl -s -o /dev/null -w '%{http_code}\n' http://juice-shop:3000
```

Create the task in the UI: **Tasks → New task**.

| Field | Value |
|---|---|
| Scope | `http://juice-shop:3000` (this address only) |
| Goal | Assess the web application for common vulnerabilities (injection, broken authentication, XSS, broken access control). Enumerate the attack surface, verify findings with concrete evidence, and produce a report. Stay strictly within scope; avoid destructive actions. |

Start it and watch the **Planner**, **Worker** and **Report writer** agents work through recon, testing, and reporting. Keep an eye on **Approval Records** — Artex pauses there for actions that need your sign-off.

---

## You're deployed

- [x] Ubuntu VM on an ArtexNet NAT network, SSH on host port 2222
- [x] Docker Engine + Compose
- [x] Artex built from source; app + Postgres containers running
- [x] Web UI reached over the SSH tunnel; admin account set
- [x] Claude Opus 5.5 connected and active across all agents
- [x] Scanners verified; first task running on an authorized target

### Everyday operations (VM · `~/ARTEX`)

| Task | Command |
|---|---|
| Status | `docker compose ps` |
| Logs (live) | `docker compose logs -f artex` |
| Stop | `docker compose down` |
| Start | `docker compose up -d` |
| Update & rebuild | `git pull && docker compose up -d --build` |
| Reset password | `./reset-password.sh` |

---

*Artex deployment runbook — VirtualBox 7.2 / Ubuntu Server 26.04 / Docker Compose / Claude Opus 5.5. Use only against systems you own or are authorized in writing to test.*
