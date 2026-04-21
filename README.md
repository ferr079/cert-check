# cert-check

[![ShellCheck](https://github.com/ferr079/cert-check/actions/workflows/lint.yml/badge.svg)](https://github.com/ferr079/cert-check/actions/workflows/lint.yml)

Lightweight TLS certificate expiration monitor for self-hosted services. Designed for homelabs using a reverse proxy (Traefik, Nginx, Caddy) with internal CAs like step-ca or Let's Encrypt.

## What it does

Checks the TLS certificate expiration date for each configured service via SNI, and reports OK / WARNING / EXPIRED status. Optional Telegram alerts via the `guardian-certs` wrapper.

```
$ cert-check all
OK authentik — 76d remaining (Jun 30 2026)
OK forgejo — 76d remaining (Jun 30 2026)
OK grafana — 76d remaining (Jun 30 2026)
OK jellyfin — 76d remaining (Jun 30 2026)

---
TOTAL: 4 checked — 4 OK, 0 WARNING, 0 CRITICAL
```

## Why

Internal CAs (step-ca, smallstep, CFSSL) issue short-lived certificates. ACME auto-renewal usually works, but when it silently fails (DNS down at boot, Traefik race condition, CA unreachable), you only find out when users hit browser warnings. This script catches it early.

## Requirements

- `openssl` (for certificate inspection)
- `curl` (for Telegram alerts, optional)
- Bash 4+ (associative arrays)

## Quick start

```bash
# Install
sudo cp cert-check /usr/local/bin/
sudo chmod +x /usr/local/bin/cert-check

# Configure
sudo mkdir -p /etc/cert-check
sudo cp services.conf.example /etc/cert-check/services.conf
# Edit with your services

# Test
cert-check all
cert-check all --warn 30
cert-check traefik forgejo
```

## Configuration

### services.conf

One service per line. Three columns:

```
service_name  hostname[:port]  [proxy_ip:proxy_port]
```

| Column | Required | Description |
|--------|----------|-------------|
| `service_name` | Yes | Label for display |
| `hostname` | Yes | FQDN (used as SNI) |
| `proxy_ip:port` | No | Reverse proxy to connect to (if services share one IP) |

**Direct check** (connect to the hostname directly):
```
mysite  mysite.example.com
```

**Via reverse proxy** (all traffic through one Traefik/Nginx IP):
```
traefik     traefik.home.lab       192.168.1.10:443
forgejo     forgejo.home.lab       192.168.1.10:443
vaultwarden vaultwarden.home.lab   192.168.1.10:443
```

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CERT_WARN_DAYS` | 14 | Warning threshold in days |
| `CERT_CHECK_CONFIG` | `/etc/cert-check/services.conf` | Config file path |

## Telegram alerts (optional)

`guardian-certs` wraps `cert-check` and sends a Telegram notification **only when something is wrong** (silent on all-OK).

```bash
sudo cp guardian-certs /usr/local/bin/
sudo chmod +x /usr/local/bin/guardian-certs
sudo cp guardian.env.example /etc/cert-check/guardian.env
sudo chmod 600 /etc/cert-check/guardian.env
# Edit with your Telegram bot token and chat ID

# Daily check at 10am
echo "0 10 * * * root /usr/local/bin/guardian-certs" | sudo tee /etc/cron.d/cert-check
```

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | All certificates OK |
| 1 | At least one expiring within warning threshold |
| 2 | At least one expired, unreachable, or unparseable |

## License

MIT
