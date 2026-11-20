# I2P Hidden Portfolio — Hardened Self-Hosting Home Lab

My portfolio site, self-hosted as a **hidden service on the I2P peer-to-peer network**, served from a **hardened rootless Podman container** on a Linux (Pop!_OS) home lab.

**Live on I2P:** `http://s7ca4oi25h4j4xyjvim7kwy6z62gukc7k7rkzxtmkul2nkdvhr4a.b32.i2p`
*(needs an I2P router; clearnet version: https://bippitybits.com)*

---

## Architecture

```
I2P network (encrypted, peer-to-peer)
        │
        ▼
 i2pd router (host, systemd service)
   └─ server tunnel "my-website" → 127.0.0.1:8080
        │
        ▼
 Podman container "i2psite" (rootless)
   └─ nginx-unprivileged → serves static site (read-only mount)
```

- No ports open to the home network or the internet — the web server listens on **localhost only**.
- Visitors reach the site only through I2P; the host's IP address is never exposed.
- Host firewall (`ufw`) blocks all incoming connections.

## Container hardening

| Control | Flag | Why |
|---|---|---|
| Rootless container | `podman` as normal user | A container escape lands as an unprivileged user, not root |
| Non-root web server | `nginx-unprivileged` image | nginx itself never runs as root |
| Read-only filesystem | `--read-only` + `--tmpfs /tmp` | Attacker can't modify the container |
| No Linux capabilities | `--cap-drop=ALL` | Removes all special kernel privileges |
| No privilege escalation | `--security-opt no-new-privileges` | Blocks setuid-style escalation |
| Resource limits | `--memory 128m --pids-limit 50` | Limits damage from abuse / fork bombs |
| Localhost-only binding | `-p 127.0.0.1:8080:8080` | Not reachable from the LAN |
| Read-only content | `-v ~/i2psite:...:ro` | Site files can't be changed from inside |
| Static site only | — | No logins, database or forms = minimal attack surface |

## Setup

See [`setup.sh`](setup.sh) — one script, safe to re-run (it won't add config twice).

## Troubleshooting log (real incident)

**Problem:** i2pd started, then crashed ~15 seconds later, every time.

**Investigation:**
```bash
systemctl is-active i2pd
sudo journalctl -u i2pd -n 15 --no-pager
```
Logs showed `Main process exited, code=dumped, status=6/ABRT` and later `status=11/SEGV` — repeated core dumps.

**Root cause:** the distribution's packaged i2pd (2.49.0) was outdated and unstable on this system.

**Fix:** switched to the official PurpleI2P repository (signed packages), upgraded, restarted, then verified the service stayed up for 60+ seconds before moving on.

**Lesson:** reproduce → read the logs → fix the cause, not the symptom → verify.

## Secrets handling

- The site's identity key (`/var/lib/i2pd/my-website.dat`) defines the `.i2p` address — whoever holds it controls the address.
- It's backed up in an **encrypted 7-Zip archive** (AES, encrypted file names) on offline drives.
- **No key material is stored in this repository.**

## Skills demonstrated

Linux administration · systemd & journalctl troubleshooting · package/repository management · rootless containers (Podman) · least-privilege hardening · reverse tunnels / hidden services · firewalling (ufw) · secrets backup · documentation
