# Surfshark WireGuard IP Monitor

Small Python utilities for tracking Surfshark WireGuard endpoint IP changes and sending update notifications through Telegram.

The repository currently contains experimental scripts. Treat it as a personal automation toolkit, not a polished package.

## What It Does

- Fetches Surfshark server location metadata.
- Builds request payloads for WireGuard-capable locations.
- Checks the current endpoint IP for each location.
- Stores the last observed IP data in `last_ips.json`.
- Sends Telegram notifications when endpoint IPs change.
- Generates WireGuard config files for changed endpoints and sends them to Telegram.

## Repository Layout

| File | Purpose |
| --- | --- |
| `main.py` | Checks Surfshark endpoint IPs and records changed locations in `last_ips.json`. |
| `wireguard.py` | Reads changed locations from `last_ips.json`, requests WireGuard configs, and sends them through Telegram. |
| `last_ips.json` | Runtime state file storing previously observed endpoint data. |

## Requirements

- Python 3.10+
- `requests`
- Telegram bot token and chat ID
- Surfshark account/session data required by the Surfshark manual WireGuard endpoints

Install the Python dependency:

```bash
python -m pip install requests
```

## Configuration

Before running the scripts, configure these values in the Python files:

| Setting | File | Notes |
| --- | --- | --- |
| `bot_token` | `main.py`, `wireguard.py` | Telegram bot token used for notifications. |
| `chat_id` | `main.py`, `wireguard.py` | Telegram chat ID that receives alerts/configs. |
| `userPrivKey` | `wireguard.py` | WireGuard private key used when generating configs. |
| `headers["Cookie"]` | `wireguard.py` | Authenticated Surfshark browser session cookie. |

Do not commit real tokens, cookies, or private keys. Prefer moving these values to environment variables before using this as a maintained automation tool.

## Usage

Run the IP monitor first:

```bash
python main.py
```

If changes are detected, `last_ips.json` is updated with entries marked as changed.

Generate and send WireGuard config files for changed endpoints:

```bash
python wireguard.py
```

Send generated configs as Telegram code blocks instead of file attachments:

```bash
python wireguard.py --code
```

## Runtime State

`last_ips.json` is used as a local cache. On first run, the monitor creates or refreshes this state based on the current Surfshark endpoint responses.

Example shape:

```json
{
  "Location Name": {
    "ip": "203.0.113.10",
    "pubKey": "server-public-key",
    "connectionName": "example-connection",
    "status": "changed-or-unchanged"
  }
}
```

## Security Notes

- Never commit Telegram bot tokens, Surfshark cookies, WireGuard private keys, or generated `.conf` files.
- Rotate any token or cookie that has ever been committed or pasted into a shared environment.
- Generated WireGuard configs contain sensitive private networking material and should be handled like secrets.
- If this repo becomes a real maintained tool, the first refactor should move all credentials to environment variables or a local ignored config file.

## Maintenance Notes

This repo was renamed from `fuckshark` to `surfshark-wireguard-ip-monitor` to make the project purpose clear and suitable for a public GitHub profile.

Suggested future cleanup:

- Replace hardcoded credentials with environment variables.
- Split network calls into testable functions.
- Add a dry-run mode that prints planned changes without sending Telegram messages.
