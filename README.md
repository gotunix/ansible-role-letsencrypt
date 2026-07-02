# Ansible Role — Let's Encrypt (Cloudflare DNS Challenge)

Install certbot on remote servers and issue TLS certificates via Let's Encrypt DNS-01 challenge through Cloudflare. Certificates are issued and renewed directly on the target hosts.

## How It Works

```
┌──────────────────────────────────────────────────────────────┐
│ On each remote host:                                         │
│                                                              │
│  certbot ──► Cloudflare DNS ──► Let's Encrypt ──► certs     │
│                                                              │
│  /etc/letsencrypt/live/ ──► /etc/ssl/certs/example.com.pem  │
│                          ──► /etc/ssl/private/example.com.key│
│                                                              │
│  cron: certbot renew (daily)                                 │
└──────────────────────────────────────────────────────────────┘
```

## Quick Start

```bash
ansible-playbook site.yaml -i inventory.ini --ask-vault-pass
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `letsencrypt_email` | `""` | ACME account email (required) |
| `letsencrypt_staging` | `false` | Use LE staging environment for testing |
| `letsencrypt_cloudflare_api_token` | `""` | Cloudflare API token — **vault this** |
| `letsencrypt_credentials_dir` | `/etc/letsencrypt` | Where Cloudflare credentials file is stored |
| `letsencrypt_domains` | `[]` | List of domain dicts (see below) |
| `letsencrypt_cert_dir` | `/etc/ssl/certs` | Destination for cert + chain files |
| `letsencrypt_key_dir` | `/etc/ssl/private` | Destination for private key |
| `letsencrypt_cert_owner` | `root` | File owner |
| `letsencrypt_cert_group` | `root` | File group |
| `letsencrypt_cert_mode` | `0644` | Cert file permissions |
| `letsencrypt_key_mode` | `0600` | Key file permissions |
| `letsencrypt_renew_cron` | `true` | Set up automatic renewal cron job |

> [!IMPORTANT]
> The Cloudflare API token needs `Zone:DNS:Edit` permission for the target zone(s).
> Create one at [Cloudflare Dashboard → API Tokens](https://dash.cloudflare.com/profile/api-tokens).

### Domain Format

```yaml
letsencrypt_domains:
  - domain: example.com
    sans:                     # optional Subject Alternative Names
      - "*.example.com"

  - domain: api.example.com
    sans: []
```

## Deployed Files

| File | Path |
|---|---|
| Full chain (cert + intermediate) | `/etc/ssl/certs/<domain>.pem` |
| Intermediate chain only | `/etc/ssl/certs/<domain>.chain.pem` |
| Private key | `/etc/ssl/private/<domain>.key` |

## Renewal

When `letsencrypt_renew_cron: true` (default), a cron job runs daily at 3:30 AM:

```
certbot renew --quiet --deploy-hook 'systemctl reload nginx || true'
```

Certbot only renews certificates within 30 days of expiry.

## Handlers

The role notifies `Certificates changed` when cert files are issued or updated. Use this in your nginx/haproxy/etc. roles:

```yaml
# In your web server role's handlers:
- name: Certificates changed
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

## Tags

| Tag | Controls |
|---|---|
| `letsencrypt` | Entire role |
| `install` | Certbot installation + credentials |
| `issue` | Certificate issuance + deployment |

## File Structure

```
letsencrypt/
├── site.yaml
└── roles/
    └── letsencrypt/
        ├── defaults/main.yaml
        ├── handlers/main.yaml
        ├── tasks/
        │   ├── main.yaml
        │   ├── install.yaml
        │   └── issue.yaml
        └── templates/
            └── cloudflare.ini.j2
```

## Requirements

- Ansible ≥ 2.9
- Target hosts running Debian/Ubuntu
- Cloudflare API token with `Zone:DNS:Edit` permission

## Security Notes

- Cloudflare credentials file is `0600`, owned by root
- Credentials directory is `0700`
- Private keys are `no_log` during deployment
- Private key files are `0600` on target hosts
- **Always store the API token in Ansible vault**
