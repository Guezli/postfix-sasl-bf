# Crowdsec scenario — `Guezli/postfix-sasl-bf` (Mailcow / postfix)

> Detect **slow / distributed** SASL LOGIN bruteforces against postfix.
> Specifically built for and tested with **[Mailcow](https://mailcow.email/)**
> running Crowdsec on the host (not inside the Mailcow stack).

> [!NOTE]
> **Now available in the official [Crowdsec Hub](https://hub.crowdsec.net/author/Guezli/configurations/postfix-sasl-bf).**
> Install with `sudo cscli scenarios install Guezli/postfix-sasl-bf` — see [Installation](#installation) below.

## Why this exists

The official [`crowdsecurity/postfix-spam`](https://hub.crowdsec.net/author/crowdsecurity/scenarios/postfix-spam) scenario is tuned for *fast* spam waves:

```yaml
capacity: 5
leakspeed: 10s   # 1 token leaks every 10 seconds
```

That works great when a botnet floods one IP at >1 attempt per 2 seconds. But many real-world bruteforces are deliberately **slow and distributed**: a /24 of IPs each tries one or two SASL LOGINs per hour, hoping to stay below per-IP thresholds. With the default leakspeed, the bucket leaks faster than it fills, so `postfix-spam` never triggers.

Mailcow's built-in F2B catches such patterns (default 20 fails / 30 min in `netfilter-mailcow`), but if you also run Crowdsec on the host you want the same protection on Layer 3 — and the bans should land in CAPI for cross-instance sharing. This scenario fills exactly that gap.

## What it does

```yaml
type: leaky
groupby: evt.Meta.source_ip
capacity: 2
leakspeed: 7200s      # 1 token leaks every 2h
blackhole: 2h
```

→ **3 SASL LOGIN failures from the same IP within ~2h** trigger a ban.

The filter matches the syslog message format that the official `crowdsecurity/postfix-logs` parser already extracts (Mailcow's postfix container produces exactly this format):

```
warning: unknown[<IP>]: SASL LOGIN authentication failed: ...
```

## Installation

### Recommended: from the official Hub

```bash
sudo cscli collections install crowdsecurity/postfix    # provides the postfix-logs parser
sudo cscli scenarios install Guezli/postfix-sasl-bf
sudo systemctl reload crowdsec
```

You still need an acquisition that reads postfix logs — see the [Requirements](#requirements) section below for examples.

### Alternative: bundled installer (auto-configures Mailcow acquisition)

If you're running Mailcow and want the acquisition snippet for the dockerized postfix container auto-detected and written in one step, this repo ships a small installer:

```bash
curl -fsSL https://raw.githubusercontent.com/Guezli/postfix-sasl-bf/main/install.sh | sudo bash
```

The script handles everything end-to-end (collection + scenario + Mailcow-or-file acquisition detection + reload) and exits when it's done. Re-run the same command anytime to refresh; the script is idempotent and safe to repeat.

### How [`install.sh`](install.sh) works

It's a ~150-line bash script that performs six steps and exits. Nothing runs in the background, nothing gets installed as a service.

| Step | What it does | If already done |
|---|---|---|
| 1 | Checks `cscli` is on PATH | aborts with error if Crowdsec isn't installed |
| 2 | Installs `crowdsecurity/postfix` collection (provides the postfix-logs parser the scenario depends on) | skipped |
| 3 | Detects a postfix log source: **Mailcow container** (`mailcowdockerized-postfix-mailcow-1`) → docker acquisition; else **/var/log/mail.log** → file acquisition; else warns | skipped if any existing acquisition already references postfix |
| 4 | Downloads `scenarios/postfix-sasl-bf.yaml` from this repo to `/etc/crowdsec/scenarios/` | skipped if file is already byte-identical |
| 5 | `systemctl reload crowdsec` | warns if Crowdsec isn't active |
| 6 | Verifies the scenario shows up in `cscli scenarios list` | aborts with error if not |

The script makes **no other changes** — it doesn't touch your existing collections, parsers, profiles, bouncers, or any non-postfix acquisitions.

### Verify it worked

```bash
sudo cscli scenarios list | grep postfix-sasl-bf
sudo cscli alerts list --scenario Guezli/postfix-sasl-bf --since 24h
```

### Manual install (if you'd rather inspect every change)

```bash
# 1. Scenario
sudo curl -fsSL -o /etc/crowdsec/scenarios/postfix-sasl-bf.yaml \
  https://raw.githubusercontent.com/Guezli/postfix-sasl-bf/main/scenarios/postfix-sasl-bf.yaml

# 2. Acquisition (Mailcow example)
sudo tee /etc/crowdsec/acquis.d/postfix-sasl-bf.yaml >/dev/null <<'EOF'
source: docker
container_name:
  - mailcowdockerized-postfix-mailcow-1
labels:
  type: syslog
EOF

# 3. Reload
sudo systemctl reload crowdsec
```

## Requirements

| Component | Source |
|---|---|
| Parser `crowdsecurity/postfix-logs` | Crowdsec hub (auto-installed with the `crowdsecurity/postfix` collection: `cscli collections install crowdsecurity/postfix`) |
| Acquisition for postfix logs | `/etc/crowdsec/acquis.yaml` must read postfix logs (file or docker) |

### Example acquisition for Mailcow's postfix container

```yaml
source: docker
container_name:
  - mailcowdockerized-postfix-mailcow-1
labels:
  type: syslog
```

For non-Mailcow postfix (file-based syslog):

```yaml
source: file
filenames:
  - /var/log/mail.log
labels:
  type: syslog
```

## Mailcow-specific notes

- **Crowdsec runs on the Mailcow host**, not inside a Mailcow container. The official Mailcow stack ships its own `netfilter-mailcow` (a Python-based F2B-like component); that is unrelated to this scenario.
- **Both can run in parallel.** Crowdsec catches the IP at Layer 3 via `nftables` *before* it reaches the Mailcow Docker network. Mailcow's F2B is the second line of defense and also catches SOGo-Webmail bruteforces (which Crowdsec doesn't cover out of the box).
- Mailcow updates may rebuild the postfix container — make sure your acquisition `container_name` still matches after a Mailcow update. With the default `mailcowdockerized` compose project name, the container is always `mailcowdockerized-postfix-mailcow-1`.

## Tuning

Defaults are deliberately permissive to avoid false positives from typo-prone real users.

If you have many internal mail clients and see legitimate retries:
- raise `leakspeed` to `3600s` (1h) — bucket leaks faster, more retries needed
- raise `capacity` to `4` — needs 5+ failures to trigger

If you want stricter (e.g. you know no internal SMTP-AUTH should ever fail):
- lower `capacity` to `1` — 2nd failure overflows
- shorten `leakspeed` to `1800s`

## False-positive considerations

- A user who genuinely typos their SMTP password 3 times within 2h **will** be banned for 2h
- Mitigation: whitelist your office IP / VPN range via `crowdsecurity/whitelists` or a custom whitelist
- For multi-tenant mailservers consider raising `capacity` to 4-5

## Pairs well with

This scenario covers slow/distributed SMTP-bruteforce, but real Mailcow setups face attacks on more layers. Companion scenarios:

- **[`Guezli/postfix-honeypot-users`](https://github.com/Guezli/postfix-honeypot-users)** — instant-bans IPs that try SASL LOGIN with well-known role addresses (`postmaster@`, `admin@`, `info@`, …) that should never be actual SMTP login accounts. Catches the **distributed wordlist-style** bruteforce where each IP only tries 1-2 attempts and slips below this scenario's threshold.
- **[`melite/dovecot-slow-bf`](https://hub.crowdsec.net/author/melite/scenarios/dovecot-slow-bf)** — same idea as this scenario but for **IMAP/POP3** (Dovecot) instead of SMTP (Postfix). Install via `cscli scenarios install melite/dovecot-slow-bf`.
- **[`melite/dovecot-time-based-bf`](https://hub.crowdsec.net/author/melite/scenarios/dovecot-time-based-bf)** — time-distributed IMAP-bruteforce variant. `cscli scenarios install melite/dovecot-time-based-bf`.
- **`crowdsecurity/postfix`** collection — the official one. Already gives you `postfix-spam` (fast-spam waves), `postfix-non-smtp-command`, `postfix-relay-denied`, `postfix-helo-rejected`, `postscreen-rbl`. `cscli collections install crowdsecurity/postfix`.

Together these cover the bulk of inbound bruteforce traffic on a typical Mailcow setup — SMTP slow & honeypot, IMAP slow & time-based, plus the official fast-pattern coverage.

## Detection background

Built and validated against Mailcow on a VPS that sees ~30 distinct slow-bruteforce IPs per day, mostly from small VPS-hosted bot networks (KOI Cloud Services, Ayosoft, DigitalOcean, etc.). The threshold math (`capacity / leakspeed`) was chosen to catch attackers averaging ≥1.5 attempts/hour per IP, while leaving headroom for clumsy humans.

## License

MIT — see [LICENSE](LICENSE).
