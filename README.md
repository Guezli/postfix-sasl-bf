# Crowdsec scenario — `Guezli/postfix-sasl-bf`

> Detect **slow / distributed** SASL LOGIN bruteforces against postfix.

## Why this exists

The official [`crowdsecurity/postfix-spam`](https://hub.crowdsec.net/author/crowdsecurity/scenarios/postfix-spam) scenario is tuned for *fast* spam waves:

```yaml
capacity: 5
leakspeed: 10s   # 1 token leaks every 10 seconds
```

That works great when a botnet floods one IP at >1 attempt per 2 seconds. But many real-world bruteforces are deliberately **slow and distributed**: a /24 of IPs each tries one or two SASL LOGINs per hour, hoping to stay below per-IP thresholds. With the default leakspeed, the bucket leaks faster than it fills, so `postfix-spam` never triggers.

Mailcow's built-in F2B catches such patterns (default 20 fails / 30 min), but if you run Crowdsec on the host you want the same protection on Layer 3 — and the bans should land in CAPI for cross-instance sharing.

## What it does

```yaml
type: leaky
groupby: evt.Meta.source_ip
capacity: 2
leakspeed: 7200s      # 1 token leaks every 2h
blackhole: 2h
```

→ **3 SASL LOGIN failures from the same IP within ~2h** trigger a ban.

The filter matches the syslog message format that the official `crowdsecurity/postfix-logs` parser already extracts:

```
warning: unknown[<IP>]: SASL LOGIN authentication failed: ...
```

## Installation

```bash
# Install
cscli scenarios install Guezli/postfix-sasl-bf

# Reload
sudo systemctl reload crowdsec
```

Or pin via [collection](#) (TBD) or manual copy:

```bash
sudo cp scenarios/postfix-sasl-bf.yaml /etc/crowdsec/scenarios/
sudo systemctl reload crowdsec
```

## Requirements

| Component | Source |
|---|---|
| Parser `crowdsecurity/postfix-logs` | hub (auto if you install `crowdsecurity/postfix` collection) |
| Acquisition for postfix logs | your `/etc/crowdsec/acquis.yaml` must read postfix logs (file or docker) |

Example acquisition for Mailcow's postfix container:

```yaml
source: docker
container_name:
  - mailcowdockerized-postfix-mailcow-1
labels:
  type: syslog
```

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
- For `Mail-in-a-Box`-style multi-tenant mailservers, consider raising `capacity` to 4-5

## Detection background

Built and validated against Mailcow on a VPS that sees ~30 distinct slow-bruteforce IPs per day, mostly from small VPS-hosted bot networks. Discussed in [Mailcow community](https://community.mailcow.email/) — happy to merge improvements.

## License

MIT — see [LICENSE](LICENSE).
