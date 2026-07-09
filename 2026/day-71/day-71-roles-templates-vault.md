# Day 71 — Ansible Roles, Templates & Vault

## 1. Jinja2 Templates (Task 1)

`templates/nginx-vhost.conf.j2`:

```jinja2
# Managed by Ansible -- do not edit manually
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

Rendered on host `web01` with `app_name: terraweek-app`, `http_port: 80`:

```nginx
# Managed by Ansible -- do not edit manually
server {
    listen 80;
    server_name web01;

    root /var/www/terraweek-app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/terraweek-app_access.log;
    error_log /var/log/nginx/terraweek-app_error.log;
}
```

Running with `--diff` shows Ansible highlighting exactly which lines changed
between the previous file on disk and the newly rendered template — every
`{{ variable }}` is replaced with its live value (Jinja2 filters like
`| default(80)` only kick in when the variable is undefined).

```
ansible-playbook template-demo.yml --diff
```

**Verification:** SSH'd into the web server and `cat`'d
`/etc/nginx/conf.d/terraweek-app.conf` — all placeholders were replaced with
real values (hostname, app name, IP-derived paths). No `{{ }}` syntax
remained in the output file.

*(Screenshot placeholder: terminal showing `--diff` output and the rendered
config on the target host.)*

---

## 2. Role Directory Structure (Task 2)

Generated with:

```
ansible-galaxy init roles/webserver
```

Resulting structure (unused dirs trimmed):

```
roles/
  webserver/
    tasks/
      main.yml         # task list executed when the role runs
    handlers/
      main.yml         # handlers triggered by notify:
    templates/
      nginx.conf.j2
      vhost.conf.j2
      index.html.j2
    files/
      (unused — no static files needed)
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml         # role metadata / Galaxy dependencies
    README.md           # auto-generated docs stub
```

### `vars/main.yml` vs `defaults/main.yml`

| | `vars/main.yml` | `defaults/main.yml` |
|---|---|---|
| Priority | High — near the top of Ansible's variable precedence | Lowest priority of all variable sources |
| Intent | Values the role author does **not** want easily overridden | Sane fallback values meant to be overridden by callers |
| Override | Hard to override (needs `-e` extra-vars or similar high-precedence source) | Easy to override — just pass `vars:` when calling the role |
| Typical use | Constants tied to the role's internal logic (package names, fixed paths) | Tunables like ports, app names, feature flags |

In short: `defaults/` is the role's public, "please override me" configuration
surface; `vars/` is closer to internal constants.

---

## 3. Custom Webserver Role (Task 3)

Final role layout:

```
roles/webserver/
  defaults/main.yml    -> http_port, app_name, max_connections
  tasks/main.yml        -> install nginx, deploy configs, start service
  handlers/main.yml     -> Restart Nginx
  templates/
    nginx.conf.j2
    vhost.conf.j2
    index.html.j2
```

`templates/vhost.conf.j2` (built on the pattern from Task 1):

```jinja2
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

`templates/nginx.conf.j2` (minimal top-level config driving worker/connection
tuning from role vars):

```jinja2
user nginx;
worker_processes auto;

events {
    worker_connections {{ max_connections | default(512) }};
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    include /etc/nginx/conf.d/*.conf;
}
```

Called from `site.yml`:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80
```

```
ansible-playbook site.yml
```

**Verification:** `curl http://<web-host-ip>/` returned the custom index
page with the app name, hostname, and IP populated by
`{{ ansible_hostname }}` / `{{ ansible_default_ipv4.address }}`, confirming
the role ran end-to-end (install → template → start → serve).

*(Screenshot placeholder: `ansible-playbook site.yml` run showing all tasks
`ok`/`changed`, plus the `curl` output.)*

---

## 4. Ansible Galaxy — Community Roles (Task 4)

Searched and installed a community role:

```
ansible-galaxy search nginx --platforms EL
ansible-galaxy search mysql
ansible-galaxy install geerlingguy.docker
ansible-galaxy list
```

Used directly in `docker-setup.yml`:

```yaml
---
- name: Install Docker using Galaxy role
  hosts: app
  become: true
  roles:
    - geerlingguy.docker
```

`requirements.yml` for pinned, repeatable installs across a mixed
group of hosts:

```yaml
---
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
  - name: geerlingguy.ntp
```

```
ansible-galaxy install -r requirements.yml
```

**Why `requirements.yml` instead of installing roles manually?**
- **Reproducibility** — anyone cloning the repo runs one command
  (`ansible-galaxy install -r requirements.yml`) instead of remembering a
  list of `ansible-galaxy install` commands.
- **Version pinning** — locking `geerlingguy.docker` to `7.4.1` protects
  against upstream breaking changes silently altering behavior on the next
  run.
- **CI/CD friendliness** — pipelines can bootstrap all dependencies in a
  single, scriptable step.
- **Team consistency** — everyone on the team installs the exact same role
  versions, avoiding "works on my machine" drift.

### Real-world run and troubleshooting

Running `docker-setup.yml` against a mixed inventory (`worker_amazon`,
`worker_redhat`, `worker-ubuntu`, `control-node-ubuntu`) surfaced two
distinct failures worth documenting:

**a) `worker_amazon` — package-manager mismatch**

```
fatal: [worker_amazon]: FAILED! => {"msg": "The Python 2 yum module is
needed for this module. If you require Python 3 support use the `dnf`
Ansible module instead."}
```

Amazon Linux 2023 dropped the `yum` Python bindings in favor of `dnf`
entirely, but `geerlingguy.docker`'s RedHat task path defaults to the `yum`
module. Fix options: update the role (`ansible-galaxy install
geerlingguy.docker --force`), pin `ansible_pkg_mgr=dnf` for that host in
inventory, or handle AL2023 with a separate `dnf`-based play.

**b) `worker-ubuntu` / `control-node-ubuntu` — Docker service fails to start**

```
fatal: [worker-ubuntu]: FAILED! => {"msg": "Unable to start service docker:
Job for docker.service failed because the control process exited with
error code."}
```

Packages installed cleanly, but `dockerd` itself failed on start on both
Ubuntu 24.04 hosts. Diagnosed by SSHing in and checking:

```
sudo systemctl status docker.service
sudo journalctl -xeu docker.service --no-pager | tail -50
sudo dockerd   # run in foreground for the real error
```

Likely culprits on Ubuntu 24.04: the default nftables iptables backend
conflicting with an older Docker build, a stale/invalid
`/etc/docker/daemon.json` from a prior install, or a leftover `containerd`
process holding the socket. Since both Ubuntu hosts failed identically, the
root cause was systemic rather than host-specific — a good reminder that
Galaxy roles still need environment-specific validation, they aren't
"install and forget."

Only `worker_redhat` completed the full play (`ok=11 changed=2 failed=0`).

*(Screenshot placeholder: full `ansible-playbook docker-setup.yml` run and
PLAY RECAP.)*

---

## 5. Ansible Vault — Encrypting Secrets (Task 5)

Workflow used:

```bash
# Create a new encrypted file (opens $EDITOR)
ansible-vault create group_vars/db/vault.yml

# Edit it later
ansible-vault edit group_vars/db/vault.yml

# View contents without decrypting to disk
ansible-vault view group_vars/db/vault.yml

# Encrypt an existing plaintext file
ansible-vault encrypt group_vars/db/secrets.yml
```

Vault file contents (before encryption):

```yaml
vault_db_password: SuperSecretP@ssw0rd
vault_db_root_password: R00tP@ssw0rd123
vault_api_key: sk-abc123xyz789
```

`cat group_vars/db/vault.yml` after encryption shows an unreadable
AES256-encrypted blob starting with `$ANSIBLE_VAULT;1.1;AES256` — confirming
the file is safe to commit to Git.

*(Screenshot placeholder: `cat group_vars/db/vault.yml` showing the
encrypted `$ANSIBLE_VAULT` header and ciphertext.)*

Ran a playbook against the vault-protected variables:

```
ansible-playbook db-setup.yml --ask-vault-pass
```

For automation, used a password file instead:

```bash
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore

ansible-playbook db-setup.yml --vault-password-file .vault_pass
```

Or configured permanently in `ansible.cfg`:

```ini
[defaults]
vault_password_file = .vault_pass
```

**Why is `--vault-password-file` better than `--ask-vault-pass` for
automated pipelines?**
- `--ask-vault-pass` blocks on an interactive terminal prompt — it has no
  way to work in a non-interactive CI/CD runner.
- `--vault-password-file` reads the password from a file (or a script that
  outputs it, e.g. pulling from a secrets manager), so pipelines can run
  unattended.
- It also keeps the password out of shell history and process listings,
  unlike passing `-e vault_password=...` directly.
- The password file itself is still sensitive — hence `chmod 600` and
  `.gitignore`, or better, generating it dynamically from a secrets
  manager (AWS Secrets Manager, Vault by HashiCorp, etc.) rather than
  storing it as a static file at all.

---

## 6. Combining Roles, Templates, and Vault (Task 6)

Final `site.yml` orchestrates all three host groups in one run:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80

- name: Configure app servers with Docker
  hosts: app
  become: true
  roles:
    - geerlingguy.docker

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Create DB config with secrets
      template:
        src: templates/db-config.j2
        dest: /etc/db-config.env
        owner: root
        mode: '0600'
```

`templates/db-config.j2`:

```jinja2
# Database Configuration -- Managed by Ansible
DB_HOST={{ ansible_default_ipv4.address }}
DB_PORT={{ db_port | default(3306) }}
DB_PASSWORD={{ vault_db_password }}
DB_ROOT_PASSWORD={{ vault_db_root_password }}
```

```
ansible-playbook site.yml
```

**Verification:** SSH'd into the `db` host and checked
`/etc/db-config.env` — `DB_HOST` and `DB_PORT` were rendered from facts and
defaults as expected, and `DB_PASSWORD` / `DB_ROOT_PASSWORD` were populated
from the vault-encrypted variables (decrypted transparently at runtime via
`--vault-password-file`). File permissions were confirmed as `600`
(`-rw-------`), so secrets are only readable by root on the target host.

---

## 7. Roles vs Playbooks vs Ad-hoc Commands — When to Use Which

| Approach | Best for | Example |
|---|---|---|
| **Ad-hoc command** (`ansible <group> -m <module> -a "..."`) | One-off, throwaway actions; quick checks or fixes on the fly | `ansible web -m shell -a "systemctl status nginx"` |
| **Playbook** | A specific, one-time or occasional workflow that doesn't need to be reused across projects | `template-demo.yml` — deploy one vhost config |
| **Role** | Reusable, shareable, well-structured automation meant to be applied across multiple playbooks/projects, with sane overridable defaults | `webserver` role reused across `web`, or shared via Galaxy |

Rule of thumb: reach for ad-hoc commands for quick investigation, playbooks
for a defined one-time procedure, and roles the moment logic needs to be
reused, shared with a team, or organized into testable, independent units
(tasks/handlers/templates/vars cleanly separated instead of one giant YAML
file).

---

## Summary

Today combined four core Ansible concepts into one working pipeline:

1. **Jinja2 templates** to generate host-specific config files dynamically.
2. **Roles** to package tasks, handlers, templates, and variables into a
   reusable, shareable unit with sensible defaults.
3. **Ansible Galaxy** to pull in battle-tested community roles
   (`geerlingguy.docker`) instead of reinventing Docker installation —
   while learning that Galaxy roles still need per-OS validation (the
   AL2023 `yum`/`dnf` mismatch and the Ubuntu `dockerd` start failure were
   good real-world reminders of that).
4. **Ansible Vault** to keep secrets encrypted at rest and inject them into
   templates transparently at runtime, with a password-file workflow suited
   to CI/CD.

#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham
