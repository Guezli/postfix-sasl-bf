# Crowdsec scenario — `Guezli/postfix-sasl-bf` (Mailcow / postfix)

> Detect **slow / distributed** SASL LOGIN bruteforces against postfix.
> Specifically built for and tested with **[Mailcow](https://mailcow.email/)**
> running Crowdsec on the host (not inside the Mailcow stack).

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

This repo is **not (yet) listed in the official Crowdsec Hub**, so `cscli scenarios install` won't find it.

### Quick install (recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/Guezli/postfix-sasl-bf/main/install.sh | sudo bash
```

The [`install.sh`](install.sh) script does, idempotent:

1. Verifies Crowdsec is installed
2. Installs the `crowdsecurity/postfix` collection if missing (provides the parser)
3. Auto-detects your postfix log source:
   - **Mailcow** running? → adds a docker-acquisition for `mailcowdockerized-postfix-mailcow-1`
   - Otherwise `/var/log/mail.log` exists? → adds a file-acquisition
   - Otherwise warns you to add one manually
4. Downloads the scenario to `/etc/crowdsec/scenarios/postfix-sasl-bf.yaml`
5. Reloads Crowdsec
6. Verifies the scenario loaded

If you already have a postfix acquisition configured, the script leaves it alone.

### Manual install

If you'd rather see what changes are made:

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

Verify:

```bash
sudo cscli scenarios list | grep postfix-sasl-bf
```

### Upgrading

Just re-run `install.sh` — it overwrites the scenario file in place and reloads Crowdsec. The acquisition is left untouched.

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

## Detection background

Built and validated against Mailcow on a VPS that sees ~30 distinct slow-bruteforce IPs per day, mostly from small VPS-hosted bot networks (KOI Cloud Services, Ayosoft, DigitalOcean, etc.). The threshold math (`capacity / leakspeed`) was chosen to catch attackers averaging ≥1.5 attempts/hour per IP, while leaving headroom for clumsy humans.

## License

MIT — see [LICENSE](LICENSE).
