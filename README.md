# Surfshark WireGuard IP Monitor

**You're in China. Surfshark stops working. The Great Firewall blocked your IP. You need a new one — fast.**

This tool watches Surfshark's WireGuard endpoints, detects when the GFW forces an IP change, and sends you a fresh working config on Telegram so you reconnect in seconds instead of hunting manually for hours.

---

## The Motive

Surfshark is one of the few VPNs that still works in China (2026). But it's a constant cat-and-mouse game:

1. The **Great Firewall** uses ML-based traffic fingerprinting to detect VPN protocols. WireGuard's distinctive 148-byte handshake is identified on the first packet — standard configurations are blocked with near-100% accuracy.
2. **Surfshark rotates IPs** to route around blocks. When an endpoint IP gets flagged, they swap to a new one.
3. **The problem**: you don't know when this happens. Your connection drops, and you have to manually check which servers still work, regenerate configs, and reconnect.

**This repo closes that gap.** It monitors endpoint IPs, detects changes the moment they happen, fetches fresh WireGuard configs from Surfshark, and pushes them to your Telegram — so you can swap and reconnect immediately.

Without this tool: drop everything, manually test servers, regenerate configs, hope it works.  
With this tool: Telegram ping with a new config → import → back online.

## The Problem

China's Great Firewall (GFW) upgraded to **ML-based traffic fingerprinting** in 2026. It now detects standard WireGuard, OpenVPN, and IKEv2 protocols with near-perfect accuracy by analyzing connection shape, packet timing, and handshake signatures (WireGuard's 148-byte handshake is identified on the first packet). Result: Surfshark IPs get **batch-blocked nightly**, connections drop unpredictably, and standard protocol configurations fail with near-100% certainty.

Surfshark's own support team [officially recommends manual WireGuard connections](https://support.surfshark.com/hc/en-us/articles/12629532662802-Connecting-from-China) as the primary workaround when the app fails in China — but you still need **fresh IPs** when endpoints get blocked.

## What This Tool Does

- **Monitors** Surfshark WireGuard endpoint IPs across all server locations.
- **Detects** IP changes (the GFW blocks an IP → Surfshark routes to a new one → the tool spots the diff).
- **Generates** fresh WireGuard `.conf` files for the new working endpoints.
- **Notifies** you via Telegram with the new config so you can swap and reconnect immediately.

This turns the cat-and-mouse game of "IP gets blocked, find a working one" from a manual hour-long chore into an automated notification.

## How It Fits Into a China VPN Strategy

Based on in-country testing from 2025-2026, reliable Surfshark access in China requires multiple tactics working together. This tool covers tactic #3 automatically:

| Tactic | What | Who handles it |
|--------|------|---------------|
| 1. Enable **NoBorders (Camouflage) Mode** in Surfshark settings | Disguises VPN traffic as ordinary HTTPS — essential against DPI | You (in app settings) |
| 2. **Manual WireGuard** when the app won't connect | Surfshark recommends this as the most reliable method | You (initial setup) |
| 3. **Rotate endpoints automatically** when IPs get blocked | This tool detects changes and sends fresh configs | This tool |
| 4. **Switch server regions** when one fails | EU/US servers often work better than Asia (HK/SG are most aggressively blocked) | You + tool reports |
| 5. **Keep a backup VPN** (ExpressVPN, Astrill) | No single VPN works 100% of the time; switch when Surfshark is fully blocked | You |

Additional real-world tips from China VPN testing:
- **WireGuard connections can hold ~47 seconds** before termination during peak hours; off-peak (2-7 AM local time) is more stable.
- **Cellular networks** show ~15% better success rates than hotel/office WiFi.
- **Port 443 or 80** is more likely to survive DPI than default WireGuard port 51820.
- Surfshark's **Macau SAR virtual server** offers lower latency than routing through Hong Kong.

## Repository Layout

| File | Purpose |
| --- | --- |
| `main.py` | Checks Surfshark endpoint IPs and records changed locations in `last_ips.json`. |
| `wireguard.py` | Reads changed locations from `last_ips.json`, requests WireGuard configs from Surfshark, and sends them through Telegram. |
| `last_ips.json` | Runtime state file storing previously observed endpoint data (IP, pubkey, connection name, change status). |

## Requirements

- Python 3.10+
- `requests`
- Telegram bot token and chat ID
- Active Surfshark subscription
- Surfshark browser session cookie (from logging into [Surfshark manual WireGuard page](https://account.surfshark.com/wireguard))

Install the dependency:

```bash
python -m pip install requests
```

## Configuration

Set these values in the Python files before running:

| Setting | File | How to get it |
|---------|------|---------------|
| `bot_token` | `main.py`, `wireguard.py` | Create a bot via [@BotFather](https://t.me/BotFather) on Telegram |
| `chat_id` | `main.py`, `wireguard.py` | Message your bot, visit `https://api.telegram.org/bot<token>/getUpdates` |
| `userPrivKey` | `wireguard.py` | Your WireGuard private key from the Surfshark manual setup page |
| `headers["Cookie"]` | `wireguard.py` | Browser cookie after logging into `account.surfshark.com/wireguard` |

**Important security rules:**
- Never commit tokens, cookies, or private keys to Git.
- Do not paste credentials into shared environments or logs.
- Add `last_ips.json` and any generated `.conf` files to `.gitignore`.
- Move all credentials to environment variables before using this in production.

## Usage

**Step 1 — Monitor for IP changes:**

```bash
python main.py
```

Scans all Surfshark WireGuard server locations and compares current endpoint IPs against the last known state. When an IP changes (meaning the old one likely got blocked), it marks that location in `last_ips.json` with `"status": "changed"`.

**Step 2 — Generate configs and send via Telegram:**

```bash
python wireguard.py
```

Reads changed locations from `last_ips.json`, requests fresh WireGuard `.conf` files from Surfshark for those endpoints, and sends them as file attachments to your Telegram chat.

To send configs as code blocks instead of file attachments (useful for copy-paste on mobile):

```bash
python wireguard.py --code
```

**Typical workflow when you lose connection:**
1. You notice Surfshark is blocked.
2. Run `python main.py` — it detects the IP change.
3. Run `python wireguard.py` — you get a fresh config on Telegram.
4. Import the new config into the WireGuard app and reconnect.

In the long run, automate this with a cron job or systemd timer:
```bash
# Every 30 minutes, check for IP changes
*/30 * * * * cd /path/to/repo && python main.py && python wireguard.py
```

## Runtime State

`last_ips.json` is automatically created/updated by `main.py`. Example structure:

```json
{
  "Hong Kong": {
    "ip": "203.0.113.10",
    "pubKey": "server-public-key",
    "connectionName": "generic-hk-001-hkg",
    "status": "changed"
  },
  "Singapore": {
    "ip": "198.51.100.20",
    "pubKey": "server-public-key-2",
    "connectionName": "generic-sg-003-sin",
    "status": "unchanged"
  }
}
```

On first run, all locations start as "unchanged" (baseline snapshot). Subsequent runs flag differences as "changed".

## Practical Tips for China Users

1. **Pre-download everything before entering China** — Surfshark's website, the WireGuard app, and this repo may all be blocked inside the GFW. Download configs and install everything while you still have open access.

2. **Focus on EU and US servers** — Counterintuitively, Asian servers (Hong Kong, Singapore, Japan) are the most aggressively monitored and blocked by the GFW. Germany, Netherlands, and US West Coast often hold connections longer.

3. **Pair this with the Surfshark app** — Use the app with NoBorders mode for casual browsing; fall back to manual WireGuard (via this tool's configs) when the app fails. Both can stop working simultaneously during crackdowns — that's when you need the backup VPN.

4. **Run it on a schedule** — The GFW performs batch IP blocks nightly (typically evening Beijing time). Running `main.py` every 30-60 minutes via cron ensures you're notified quickly when Surfshark routes around a block.

5. **Have a backup VPN installed** — Even the best setup can fail during GFW upgrades. Install a second VPN (ExpressVPN or Astrill) on your devices before entering China as a fallback.

6. **Consider multi-hop** — Surfshark's MultiHop servers route through two countries, making traffic pattern analysis harder for ML-based detection. If single-hop WireGuard keeps failing, try locations that support MultiHop.

## Security Notes

- Telegram bot tokens, Surfshark session cookies, and WireGuard private keys are **high-value secrets** — if exposed, an attacker can decrypt your VPN traffic or control your Telegram bot.
- Generated `.conf` files contain your private key and should never be committed or shared beyond the intended device.
- If you ever committed credentials to this repo, **rotate them immediately** — assume they are compromised.
- For production use, refactor credentials into environment variables or a local `.env` file excluded from Git.

## Potential Improvements

- Move all credentials to environment variables (no hardcoded values in Python files).
- Add Docker setup for headless server deployment.
- Support multiple notification channels (email, webhook, Signal) beyond Telegram.
- Add WireGuard config validation before sending (check if the new endpoint is actually reachable).
- Auto-apply the new config and test connectivity before notifying.
- Add dry-run mode: log planned changes without sending Telegram messages.
- Split network calls into testable functions with proper error handling.

---

*This repo was renamed from `fuckshark` to `surfshark-wireguard-ip-monitor` for professionalism. The original name reflected the frustration of getting blocked. The new name reflects the solution.*
