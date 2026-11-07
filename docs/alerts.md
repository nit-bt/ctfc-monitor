# Alerts (ctf-alert)

Runs on every platform / challenge server, checks every minute, and messages Discord / Telegram.
What to do when a message arrives: **ctf-infra** `docs/emergency.md`.

## Set up (once per server)

```bash
sudo git clone https://github.com/nit-bt/ctfc-monitor.git /opt/ctfc-monitor 2>/dev/null || sudo git -C /opt/ctfc-monitor pull
cd /opt/ctfc-monitor && sudo ./alerts/install-alerts.sh platform     # or: chal
sudo nano /etc/default/ctf-alert                     # Telegram and/or Discord, PEERS
sudo ctf-alert status                                # what it sees right now
sudo ctf-alert test                                  # phone should buzz
sudo systemctl enable --now ctf-alert.timer          # check every minute from now on
```
Telegram: talk to **@BotFather** → `/newbot` → copy the token. Send your new bot any message, open `https://api.telegram.org/bot<TOKEN>/getUpdates`, copy the number after `"chat":{"id":`. Use a group chat id instead to alert the whole team.
Discord: channel → Edit Channel → Integrations → Webhooks → New Webhook → Copy URL.

`PEERS` makes servers watch each other, so a server that dies completely still gets reported:
- platform: `PEERS="chal-1=<chal-1 ip>:2376 chal-2=<chal-2 ip>:2376"`
- each challenge server: `PEERS="platform=<platform ip>:80"`

> ⚠️ A peer address is hard-coded. If you move or retire the peer (e.g. the platform moves from GCP to Contabo, or a
> GCP VM is stopped), update `PEERS` on every server that lists it, or clear it (`PEERS=""`). A stale peer makes the
> server report **down** even though it is healthy. If each server already pushes to Kuma (`KUMA_PUSH_URL`), Kuma alerts
> on a dead server on its own, so `PEERS` is optional and `PEERS=""` is the safe default.

What it checks every minute: Docker running, RAM and disk below 85 %, containers killed for hitting their RAM cap; platform: site answers and the 4 CTFd containers run; challenge servers: firewall on and Docker API answering; plus the peers.
You get 🔴 once, a reminder every 15 min while it stays broken, ✅ when it's fixed.

**Live status in Discord:** set `DISCORD_STATUS_WEBHOOK` too and each server keeps one message up to date (RAM, disk, instances), see [monitoring.md](monitoring.md#4c-live-status-in-discord-optional).

**Dashboard:** with Uptime Kuma set up ([monitoring.md](monitoring.md)), add `KUMA_PUSH_URL` to the same config and every check shows up there too.

**Event day:** check more often, on every server: `sudo ctf-alert every 20` (and set each Push monitor in Kuma to Heartbeat Interval 40). After the event: `sudo ctf-alert every 60` (Kuma back to 90).

