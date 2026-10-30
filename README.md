# role_gitea

Install and configure [Gitea](https://gitea.io/) — a self-hosted Git service.

## Features

- Binary installation from official Gitea releases with SHA256 + GPG signature verification
- Automatic secret generation (SECRET_KEY, INTERNAL_TOKEN, JWT secrets)
- Comprehensive `app.ini` configuration via Jinja2 template (30+ sections)
- **LDAP authentication** — supports bind DN search, LDAP-simple, Active Directory, OpenLDAP
- **Admin user creation** and **local user management** via Gitea CLI
- **Custom branding** — logos, favicons, footer templates, CSS themes, public assets, custom templates
- Systemd service with **comprehensive security hardening** (17+ directives) and optional **cgroup resource limits**
- Scheduled backup via systemd timer with configurable retention
- Direct TLS termination in Gitea (certificate deployment, optional HTTP redirect)
- Database-aware restore from a `gitea dump` archive
- Fail2ban integration for brute-force protection
- Multi-OS support: Ubuntu 22.04/24.04, Debian 11/12, AlmaLinux 9/10, RockyLinux 10
- Database support: SQLite3 (default), PostgreSQL, MySQL/MariaDB
- Full app.ini coverage: UI/themes, webhooks, CORS, metrics, API, OpenID, packages, federation, migrations, mirrors, proxy, cron, markup, SSH, git timeouts
- Extensible via `gitea_extra_config` dict for arbitrary app.ini sections

## Requirements

- Ansible >= 2.11
- `role_base` (applied before this role)
- Target OS: Debian/Ubuntu or RHEL/AlmaLinux/Rocky

### Optional

- PostgreSQL or MySQL/MariaDB (if not using SQLite3)
- fail2ban (if `gitea_fail2ban_enabled: true`)
- LDAP server (if `gitea_ldap_sources` is configured)

## Role Variables

### Required

| Variable | Type | Description |
|----------|------|-------------|
| `gitea_version` | str | Gitea version to install (e.g., `"1.23.4"`) |

### Core Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_user` | `gitea` | System user |
| `gitea_group` | `gitea` | System group |
| `gitea_app_name` | `Gitea: Git with a cup of tea` | Application name in title/footer |
| `gitea_dir` | `/opt/gitea` | Base installation directory |
| `gitea_config_dir` | `/etc/gitea` | Configuration directory |
| `gitea_data_dir` | `/var/lib/gitea` | Data directory |
| `gitea_log_dir` | `/var/log/gitea` | Log directory |

### Server

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_tls_enabled` | `false` | Terminate TLS in Gitea (`PROTOCOL = https`) |
| `gitea_server_protocol` | `https` when TLS is enabled, otherwise `http` | Protocol (http, https, fcgi, unix) |
| `gitea_server_domain` | `localhost` | Domain name |
| `gitea_server_http_addr` | `0.0.0.0` | Listen address |
| `gitea_server_http_port` | `3000` | Listener port (HTTPS port when TLS is enabled) |
| `gitea_server_cert_file` | `/etc/gitea/ssl/<domain>.crt` | TLS certificate path |
| `gitea_server_key_file` | `/etc/gitea/ssl/<domain>.key` | TLS private key path |
| `gitea_tls_certificate_src` | `""` | Controller-side certificate source (optional) |
| `gitea_tls_key_src` | `""` | Controller-side key source (optional) |
| `gitea_server_redirect_http` | `true` when TLS is enabled | Redirect plain HTTP to HTTPS |
| `gitea_server_redirect_http_port` | `80` | Redirect listener port |
| `gitea_server_ssh_port` | `22` | SSH port |
| `gitea_server_offline_mode` | `false` | Disable CDN/external resources |

### TLS

With `gitea_tls_enabled: true` Gitea serves HTTPS itself, so no reverse proxy is
required. The certificate pair comes from either `gitea_tls_certificate_src` /
`gitea_tls_key_src` (controller files, deployed as `0644` certificate and
`0640` key owned by `root:gitea`), or from an internal CA role that already
placed it at `gitea_server_cert_file` / `gitea_server_key_file`. The chain is
served as installed; intermediate certificates must be concatenated into the
certificate file.

Plain HTTP is redirected to HTTPS by default when TLS is enabled. Port `80`
needs `gitea_systemd_cap_net_bind_service: true`; set a different
`gitea_server_redirect_http_port` when running unprivileged.

```yaml
gitea_tls_enabled: true
gitea_server_domain: git.example.com
gitea_server_root_url: "https://git.example.com/"
gitea_tls_certificate_src: files/gitea/git.example.com.crt
gitea_tls_key_src: files/gitea/git.example.com.key
gitea_server_redirect_http: true
gitea_systemd_cap_net_bind_service: true
```

When TLS is disabled the instance serves plain HTTP on
`gitea_server_http_port`. In that mode **a TLS-terminating reverse proxy is
required** in front of Gitea (for example `role_nginx` or `role_haproxy`);
exposing this listener directly sends credentials and session cookies in clear
text.

### Database

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_db_type` | `sqlite3` | Database type (sqlite3, mysql, postgres) |
| `gitea_db_host` | `127.0.0.1:3306` | Database host:port |
| `gitea_db_name` | `gitea` | Database name |
| `gitea_db_user` | `gitea` | Database user |
| `gitea_db_password` | `""` | Database password |

### Backup and Restore

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_backup_enabled` | `false` | Enable the systemd backup timer |
| `gitea_backup_dir` | `/opt/gitea/backups` | Archive directory |
| `gitea_backup_type` | `zip` | Archive type |
| `gitea_backup_retention_days` | `30` | Delete archives older than this (minimum 1) |
| `gitea_restore_enabled` | `false` | Restore during the role run |
| `gitea_restore_archive` | `""` | Absolute path to a `gitea dump` archive |
| `gitea_restore_force` | `false` | Allow restoring over existing repositories |
| `gitea_restore_backup_current` | `true` | Keep a copy of the current SQLite database |

`gitea dump` always writes a database dump (`gitea-db.sql`) plus repositories,
data, custom files and `app.ini`; the restore path restores all of them, so
users, repositories and settings come back together. Restore refuses to run
over an existing instance unless `gitea_restore_force: true`; for PostgreSQL
and MySQL the matching client (`psql` / `mysql`) must be installed.

### LDAP Authentication

Configure one or more LDAP sources via `gitea_ldap_sources` list. Each item supports:

| Key | Required | Description |
|-----|----------|-------------|
| `name` | yes | Display name for the auth source |
| `type` | no | `ldap` (default) or `ldap-simple` |
| `host` | yes | LDAP server hostname |
| `port` | no | Port (default: 636 for ldaps, 389 otherwise) |
| `encryption` | no | `disable`, `starttls`, or `ldaps` |
| `bind_dn` | yes (ldap) | Bind DN for search |
| `bind_password` | yes (ldap) | Bind password |
| `user_dn` | yes (ldap-simple) | DN template with `%s` for username |
| `user_search_base` | yes | Base DN for user search |
| `user_filter` | yes | LDAP filter (use `%s` for username) |
| `admin_filter` | no | Filter to grant admin privileges |
| `username_attribute` | no | Default: `uid` |
| `firstname_attribute` | no | Default: `givenName` |
| `surname_attribute` | no | Default: `sn` |
| `email_attribute` | no | Default: `mail` |
| `public_ssh_key_attribute` | no | Attribute for SSH public keys |
| `avatar_attribute` | no | Attribute for avatar images |
| `skip_tls_verify` | no | Skip TLS certificate verification |
| `sync_users` | no | Enable periodic user synchronization |
| `active` | no | Enable this source (default: true) |

### Admin & Local Users

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_admin_user` | `{}` | Admin user dict: `username`, `password`, `email` |
| `gitea_local_users` | `[]` | List of local users to create/remove |

### UI & Themes

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_ui_themes` | `gitea-auto,gitea-light,gitea-dark` | Available themes |
| `gitea_ui_default_theme` | `gitea-auto` | Default theme |
| `gitea_ui_reactions` | `+1,-1,laugh,...` | Available reaction emojis |
| `gitea_ui_max_display_file_size` | `8388608` | Max file size for inline display |

### Custom Branding

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_custom_logo_svg` | `""` | Path to custom logo.svg |
| `gitea_custom_logo_png` | `""` | Path to custom logo.png |
| `gitea_custom_favicon_svg` | `""` | Path to custom favicon.svg |
| `gitea_custom_favicon_png` | `""` | Path to custom favicon.png |
| `gitea_custom_apple_touch_icon` | `""` | Path to custom apple-touch-icon.png |
| `gitea_custom_footer_template` | `""` | Path to custom footer template |
| `gitea_custom_themes` | `[]` | CSS themes to download: `[{name, url}]` |
| `gitea_custom_public_files_src` | `""` | Directory to copy to public/assets/ |
| `gitea_custom_templates_src` | `""` | Directory to copy to templates/ |

### Features

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_mailer_enabled` | `false` | Enable email notifications |
| `gitea_actions_enabled` | `false` | Enable Gitea Actions (CI/CD) |
| `gitea_backup_enabled` | `false` | Enable scheduled backups |
| `gitea_fail2ban_enabled` | `false` | Enable fail2ban integration |
| `gitea_cors_enabled` | `false` | Enable CORS headers |
| `gitea_metrics_enabled` | `false` | Enable Prometheus metrics |
| `gitea_federation_enabled` | `false` | Enable ActivityPub federation |
| `gitea_packages_enabled` | `true` | Enable package registry |
| `gitea_openid_enable_openid_signin` | `false` | Enable OpenID sign-in |
| `gitea_proxy_enabled` | `false` | Enable outbound proxy |
| `gitea_cron_enabled` | `true` | Enable cron jobs |
| `gitea_markup_math_enabled` | `true` | Enable math rendering |
| `gitea_extra_config` | `{}` | Extra app.ini sections (dict of dicts) |

### Systemd Hardening

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_systemd_hardening_enabled` | `true` | Enable security hardening directives |
| `gitea_systemd_cap_net_bind_service` | `false` | Allow binding to ports < 1024 |
| `gitea_systemd_cgroup_enabled` | `false` | Enable cgroup resource limits |
| `gitea_systemd_cpu_quota` | `80%` | CPU quota |
| `gitea_systemd_memory_max` | `4G` | Memory limit |
| `gitea_systemd_io_weight` | `80` | IO weight |

### Firewall

| Variable | Default | Description |
|----------|---------|-------------|
| `gitea_firewall_enabled` | `true` | Re-enable firewalld on the Gitea host |
| `gitea_allowed_source_ips` | `["127.0.0.1/32"]` | IPv4 CIDRs allowed to reach Gitea and the redirect port |

See [defaults/main.yml](defaults/main.yml) for the complete list of 100+ variables.

## Dependencies

- `role_base` — OS hardening, packages, hostname configuration

## Example Playbooks

### Basic (SQLite3, HTTP)

```yaml
---
- name: Install and Configure Gitea
  hosts: gitea
  become: true
  become_method: sudo
  gather_facts: true
  roles:
    - role: role_base
      tags: base
    - role: role_gitea
      tags: gitea
      vars:
        gitea_version: "1.23.4"
        gitea_server_domain: "git.example.com"
```

### Production with LDAP and Custom Branding

```yaml
---
- name: Install and Configure Gitea
  hosts: gitea
  become: true
  become_method: sudo
  gather_facts: true
  vars_files:
    - vault/gitea.yml
  roles:
    - role: role_base
      tags: base
    - role: role_gitea
      tags: gitea
      vars:
        gitea_version: "1.23.4"
        gitea_app_name: "ACME Git Portal"
        gitea_server_domain: "git.acme.com"
        gitea_server_root_url: "https://git.acme.com/"

        # Database
        gitea_db_type: postgres
        gitea_db_host: "127.0.0.1:5432"
        gitea_db_name: gitea
        gitea_db_user: gitea
        gitea_db_password: "{{ vault_gitea_db_password }}"

        # Admin user
        gitea_admin_user:
          username: gitea_admin
          password: "{{ vault_gitea_admin_password }}"
          email: admin@acme.com

        # LDAP — Active Directory
        gitea_ldap_sources:
          - name: "ACME Active Directory"
            type: ldap
            host: "dc01.acme.com"
            port: 636
            encryption: ldaps
            bind_dn: "CN=Gitea Service,OU=Service Accounts,DC=acme,DC=com"
            bind_password: "{{ vault_gitea_ldap_bind_password }}"
            user_search_base: "DC=acme,DC=com"
            user_filter: "(&(objectCategory=Person)(sAMAccountName=%s)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))"
            admin_filter: "(memberOf=CN=Gitea Admins,OU=Groups,DC=acme,DC=com)"
            username_attribute: sAMAccountName
            firstname_attribute: givenName
            surname_attribute: sn
            email_attribute: mail
            sync_users: true
            active: true

        # Disable self-registration (LDAP only)
        gitea_service_disable_registration: true
        gitea_service_allow_only_external_registration: true

        # Custom branding
        gitea_custom_logo_svg: "files/gitea/acme-logo.svg"
        gitea_custom_favicon_png: "files/gitea/acme-favicon.png"
        gitea_custom_footer_template: "files/gitea/acme_footer.tmpl"
        gitea_custom_themes:
          - name: acme-dark
            url: "https://cdn.acme.com/gitea/acme-dark.css"
        gitea_ui_themes: "gitea-auto,gitea-light,gitea-dark,acme-dark"
        gitea_ui_default_theme: "acme-dark"

        # Features
        gitea_mailer_enabled: true
        gitea_mailer_smtp_addr: "smtp.acme.com"
        gitea_mailer_smtp_port: 587
        gitea_mailer_user: "gitea@acme.com"
        gitea_mailer_password: "{{ vault_gitea_mailer_password }}"
        gitea_mailer_from: "Gitea <gitea@acme.com>"
        gitea_backup_enabled: true
        gitea_fail2ban_enabled: true
        gitea_actions_enabled: true
        gitea_metrics_enabled: true
        gitea_cron_enabled: true

        # Systemd hardening with resource limits
        gitea_systemd_cgroup_enabled: true
        gitea_systemd_memory_max: "8G"
        gitea_systemd_cpu_quota: "200%"

        # Webhook and API tuning
        gitea_webhook_deliver_timeout: 30
        gitea_api_max_response_items: 100
```

### OpenLDAP Example

```yaml
gitea_ldap_sources:
  - name: "Corporate OpenLDAP"
    type: ldap
    host: "ldap.example.com"
    port: 389
    encryption: starttls
    bind_dn: "cn=readonly,dc=example,dc=com"
    bind_password: "{{ vault_gitea_ldap_bind_password }}"
    user_search_base: "ou=people,dc=example,dc=com"
    user_filter: "(&(objectClass=posixAccount)(uid=%s)(memberOf=cn=Gitea Users,ou=groups,dc=example,dc=com))"
    admin_filter: "(memberOf=cn=admins,ou=groups,dc=example,dc=com)"
    username_attribute: uid
    firstname_attribute: givenName
    surname_attribute: sn
    email_attribute: mail
    public_ssh_key_attribute: sshPublicKey
    sync_users: true
```

### LDAP Simple (Direct Bind)

```yaml
gitea_ldap_sources:
  - name: "Simple LDAP"
    type: ldap-simple
    host: "ldap.example.com"
    port: 389
    encryption: starttls
    user_dn: "uid=%s,ou=people,dc=example,dc=com"
    user_filter: "(objectClass=posixAccount)"
    user_search_base: "ou=people,dc=example,dc=com"
    username_attribute: uid
    email_attribute: mail
```

## Molecule Testing

```bash
cd roles/role_gitea
molecule test                        # Default scenario (RockyLinux 9)
molecule test -s almalinux-9         # Specific distro
molecule test -s ubuntu-2404         # Ubuntu 24.04
molecule converge                    # Run converge only
molecule verify                      # Run verify only
```

## License

MIT

## Author

gkorkmaz
