# Ansible Role: certbot

[![CI](https://github.com/tjg-homelab/ansible-role-certbot/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-certbot/actions/workflows/ci.yml)

Installs [certbot](https://certbot.eff.org/) (via snap — certbot's recommended
method) and sets a host up to **issue and auto-renew its own** Let's Encrypt
certificate, with a systemd renewal timer.

**Scope:** this role is the *issuer*. It's the counterpart to a distribute-only
certs role — instead of one host renewing and copying the cert everywhere, **every
TLS-terminating host owns its certificate end-to-end**. Nothing depends on a central
host, and nothing depends on Ansible being re-run to keep the cert fresh: once
provisioned, the on-host timer renews it. Private keys are generated locally and
never leave the host.

## What this role does

- Installs certbot via snap (`core` → `certbot --classic`), and for DNS-01 the
  `certbot-dns-cloudflare` plugin snap (trusted + connected)
- Optionally writes a Cloudflare DNS credentials file (DNS-01)
- Performs a **guarded initial issuance** (`certbot certonly`) — skipped when the
  certificate lineage already exists, so it's safe to re-run and safe on hosts that
  already hold the cert. Runs `certbot_deploy_hook` immediately after a successful
  issuance (same as a renewal does), so a web server serving a bootstrap
  placeholder — or nothing — picks up the real cert right away instead of waiting
  for the next scheduled renewal
- Installs a `certbot-renew.service` + `.timer` (twice-daily, randomized, persistent)
  and enables it

It does **not** configure your web server. Point nginx/apache at the cert in
`/etc/letsencrypt/live/<cert-name>/`.

## Challenge types

| `certbot_challenge` | Validation | Use case |
|---|---|---|
| `dns-cloudflare` | DNS-01 via the Cloudflare plugin | **Wildcards**; needs a scoped Cloudflare API token. Best kept to a LAN/trusted host. |
| `webroot` | HTTP-01 from `certbot_webroot_path` | A public edge that serves `:80`; **no DNS token required**. Specific names only (no wildcards). |

## Requirements

- Debian 12/13 or Ubuntu 22.04/24.04, with `snapd` (installed by the role)
- For `dns-cloudflare`: a Cloudflare API token with DNS edit on the zone (vault it)

## Key Role Variables

| Variable | Default | Description |
|---|---|---|
| `certbot_domains` | `[]` | **Required.** Cert names; first entry is the lineage name. |
| `certbot_challenge` | `dns-cloudflare` | `dns-cloudflare` \| `webroot` |
| `certbot_email` | `""` | Empty ⇒ register without email |
| `certbot_dns_credentials_content` | `""` | Cloudflare creds body (vault it); written to `certbot_dns_credentials_path` |
| `certbot_webroot_path` | `/var/www/html` | HTTP-01 webroot |
| `certbot_manage_install` | `true` | Install certbot via snap (set false in containers) |
| `certbot_manage_certificates` | `true` | Run the guarded initial issuance |
| `certbot_renew_oncalendar` | `*-*-* 03,15:00:00` | Renewal timer schedule |
| `certbot_deploy_hook` | `""` | Optional post-renew command (e.g. `systemctl reload nginx`) |

See `defaults/main.yml` for the full set.

## Example Playbooks

DNS-01 wildcard (LAN reverse proxy):

```yaml
- hosts: cardinal
  roles:
    - role: certbot
      vars:
        certbot_domains: ["folden-nissen.com", "*.folden-nissen.com"]
        certbot_challenge: dns-cloudflare
        certbot_dns_credentials_content: "dns_cloudflare_api_token = {{ vault_cloudflare_token }}"
```

HTTP-01 webroot (public edge, no DNS token):

```yaml
- hosts: donkey
  roles:
    - role: certbot
      vars:
        certbot_domains: ["www.folden-nissen.com", "blackstar.folden-nissen.com"]
        certbot_challenge: webroot
        certbot_webroot_path: /var/www/html
        certbot_deploy_hook: "systemctl reload nginx"
```

## Testing

Molecule (Docker driver) verifies the config surface — credentials file, renewal
units, and timer enablement — against Debian 12, Debian 13, and Ubuntu 24.04. snapd,
certbot, and Let's Encrypt don't run inside a CI container, so the snap install and
the real `certonly` issuance are skipped there (`certbot_manage_install: false`,
`certbot_manage_certificates: false`).

```bash
pip install ansible-core molecule molecule-plugins[docker] docker
molecule test
```

## License

MIT

## Author

Rodney Nissen ([The Jira Guy](https://thejiraguy.com)) — Senior Atlassian
Consultant & Jira Architect.
