# HTTPS for the monitor: kuma.ctfc.live + live.ctfc.live

The domain itself (Namecheap → Cloudflare, SSL mode Full (strict)) is set up in **ctf-infra** `docs/domain.md`, step 1.
This page is only the monitor VM's part.

## 1. DNS records for the monitor

Cloudflare → DNS → Records → Add record (twice):

| Type | Name | IPv4 | Proxy |
|---|---|---|---|
| A | `kuma` | monitor VM's **external** IP | 🟠 Proxied |
| A | `live` | monitor VM's **external** IP | 🟠 Proxied |

## 2. Open 80 and 443 on the monitor VM

GCP → VPC network → Firewall → rule `kuma` → Edit → TCP ports `80,443,3001,3002` → Save.
(3001/3002 get closed again in step 4.)

## 3. HTTPS on the monitor VM

**a. Origin certificate (once, free, valid 15 years):** Cloudflare → `ctfc.live` → **SSL/TLS** → **Origin Server** →
**Create Certificate** → keep the defaults (RSA, `*.ctfc.live` + `ctfc.live`, 15 years) → **Create**.
It shows two boxes. On the monitor VM:
```bash
sudo mkdir -p /etc/caddy
sudo nano /etc/caddy/origin.pem     # paste the "Origin Certificate" box, save
sudo nano /etc/caddy/origin.key     # paste the "Private Key" box, save (shown only once)
```
**b. Install Caddy + config:**
```bash
cd /opt/ctfc-monitor && sudo git pull
sudo ./https/setup-https.sh ctfc.live
```
**c.** Cloudflare → SSL/TLS → Edge Certificates: the Universal certificate shows **Active** (5-15 min after the
domain goes active). Then **https://kuma.ctfc.live** and **https://live.ctfc.live** work (same logins as before).

The same origin certificate covers `internal.ctfc.live` and `atc.ctfc.live` later (it's `*.ctfc.live`):
copy the two files to the platform's nginx.

Kuma → Settings → General → **Primary Base URL** = `https://kuma.ctfc.live` (links in notifications).

## 4. Close the raw ports

Everything now goes through HTTPS, so the old `http://IP:3001` / `:3002` should stop being reachable from the internet.

1. GCP firewall rule `kuma` → TCP ports `80,443` only → Save.
2. Before closing, point every server at the HTTPS names (on each server):
   ```bash
   sudo sed -i 's#http://<monitor-ip>:3001#https://kuma.ctfc.live#' /etc/default/ctf-alert
   sudo sed -i 's#^LIVE_URL=.*#LIVE_URL=https://live.ctfc.live#' /etc/default/ctf-live
   sudo systemctl restart ctf-live-agent && sudo ctf-alert test
   ```
3. Board → Servers tab → Board address: `https://live.ctfc.live` (new VMs get it in their install command).

> ⚠️ Do steps 2-3 on **every** server (platform and each challenge server) **before** closing the ports in step 1.
> Once the raw ports are shut, the only way in is HTTPS through Cloudflare, and Cloudflare **blocks an out-of-date agent**
> as a bot (the board goes "NO DATA", the agent logs nothing on the old version). So first update each agent:
> ```bash
> sudo git -C /opt/ctfc-monitor pull        # clone it there first if the folder is missing
> sudo cp /opt/ctfc-monitor/live/agent.py "$(systemctl cat ctf-live-agent | sed -n 's/^ExecStart=.* //p')"
> sudo systemctl restart ctf-live-agent
> ```
> After closing the ports, confirm every Kuma monitor and every board card is green. `curl -s "<KUMA_PUSH_URL>&status=up&msg=test&ping=1"` must return `{"ok":true}` from each server.

## Optional: login wall for Kuma and the board

Cloudflare → Zero Trust (free up to 50 users) → Access → Applications → Add → Self-hosted →
`kuma.ctfc.live` and `live.ctfc.live` → policy: emails of the organizers. Anyone else can't even reach the login page.
Don't do this for `internal.ctfc.live` (players) or `chal1` (not proxied).
