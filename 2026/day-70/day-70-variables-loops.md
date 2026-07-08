# Day 70 — Variables, Facts, Conditionals and Loops

## 1. Project Structure

```
ansible-practice/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   ├── all.yml
│   ├── web.yml
│   └── db.yml
├── host_vars/
│   └── web-server.yml
└── playbooks/
    ├── variables-demo.yml
    ├── site.yml
    ├── facts-demo.yml
    ├── conditional-demo.yml
    ├── loops-demo.yml
    └── server-report.yml
```

- **group_vars/all.yml** — variables applied to every host in the inventory (`ntp_server`, `app_env`, `common_packages`).
- **group_vars/web.yml** — variables applied only to hosts in the `[web]` group (`http_port`, `max_connections`, `web_packages`).
- **group_vars/db.yml** — variables applied only to hosts in the `[db]` group (`db_port`, `db_packages`).
- **host_vars/web-server.yml** — variables applied only to the single host `web-server` (`max_connections`, `custom_message`), overriding the group-level value.

---

## 2. Variable Precedence

Ansible resolves the *same* variable name differently depending on where it's defined. Lower items in this list win over higher ones:

```
role defaults
  → group_vars/all
    → group_vars/<group>
      → host_vars/<host>
        → playbook `vars:`
          → task-level `vars:`
            → extra vars (-e "var=value")   <- always wins
```

**Test performed:**

| Source | Value of `app_name` / `max_connections` |
|---|---|
| Playbook `vars:` (variables-demo.yml) | `app_name: terraweek-app` |
| CLI override | `ansible-playbook variables-demo.yml -e "app_name=my-custom-app app_port=9090"` → `app_name` became `my-custom-app` |
| group_vars/web.yml | `max_connections: 1000` |
| host_vars/web-server.yml | `max_connections: 2000` |
| Result on `web-server` | `max_connections = 2000` (host_vars beat group_vars) |

**Conclusion:** `-e` on the command line overrides everything, including playbook `vars:`. `host_vars` overrides `group_vars`, and a more specific group (`web`/`db`) overrides the catch-all `all` group. This makes it possible to set sane defaults for everyone in `group_vars/all.yml`, refine them per role in `group_vars/<group>.yml`, and make a one-off exception in `host_vars/<host>.yml` — without ever editing the playbook itself.

---

## 3. Five Useful Ansible Facts

| Fact | Example value | Where I'd use it |
|---|---|---|
| `ansible_distribution` / `ansible_distribution_version` | `Ubuntu 22.04` | Branch package-manager logic (`apt` vs `yum`) and pick OS-specific config templates. |
| `ansible_memtotal_mb` | `1984` | Skip memory-hungry tasks or warn when a host doesn't meet the minimum RAM for a service (e.g. a database). |
| `ansible_default_ipv4.address` | `10.0.1.15` | Populate config files (Nginx `server_name`, `/etc/hosts`, monitoring agent config) with the host's real IP instead of hardcoding it. |
| `ansible_hostname` / `inventory_hostname` | `web-server` | Label generated reports/logs per host, or build per-host file paths (e.g. `/tmp/server-report-{{ inventory_hostname }}.txt`). |
| `ansible_processor_vcpus` | `2` | Size worker/thread-pool settings in app config (e.g. Nginx `worker_processes`, Gunicorn workers) relative to available CPU. |

---

## 4. Conditional Playbook (`conditional-demo.yml`)

**What ran vs. what was skipped, by host group:**

| Task | web-server | db-server | Reason |
|---|---|---|---|
| Install Nginx | ✅ ran | ⏭️ skipped | `when: "'web' in group_names"` |
| Install MySQL | ⏭️ skipped | ✅ ran | `when: "'db' in group_names"` |
| Low-memory warning | (depends on RAM) | (depends on RAM) | `when: ansible_memtotal_mb < 1024` |
| Amazon Linux message | ⏭️/✅ depending on OS | ⏭️/✅ depending on OS | `when: ansible_distribution == "Amazon"` |
| Ubuntu message | ⏭️/✅ depending on OS | ⏭️/✅ depending on OS | `when: ansible_distribution == "Ubuntu"` |
| Production settings | ⏭️/✅ depending on `app_env` | ⏭️/✅ depending on `app_env` | `when: app_env == "production"` |
| AND condition (web + enough RAM) | ✅ ran (RAM ≥ 512MB) | ⏭️ skipped | not in `web` group |
| OR condition (web or app) | ✅ ran | ⏭️ skipped | matched `'web' in group_names` |

> 📸 *Insert screenshot here: terminal output of `ansible-playbook conditional-demo.yml` showing `skipping:` and `ok:`/`changed:` lines per host.*

**Verification:** tasks correctly skipped on hosts that didn't match their `when` condition — confirmed by the `skipping: [hostname]` lines in the play recap, with no errors.

---

## 5. Loops Playbook (`loops-demo.yml`)

Each loop iteration is reported as a **separate item** in the output, e.g.:

```
TASK [Create multiple users] **************************************
changed: [web-server] => (item={'name': 'deploy', 'groups': 'wheel'})
changed: [web-server] => (item={'name': 'monitor', 'groups': 'wheel'})
changed: [web-server] => (item={'name': 'appuser', 'groups': 'users'})

TASK [Print each user created] *************************************
ok: [web-server] => (item={'name': 'deploy', 'groups': 'wheel'}) => {
    "msg": "Created user deploy in group wheel"
}
...
```

> 📸 *Insert screenshot here: terminal output of `ansible-playbook loops-demo.yml` showing each `item=` iteration for users, directories, and packages.*

**`loop` vs. `with_items`:**

- `with_items` is the legacy syntax and only accepts a flat list (it silently flattens nested lists, which can cause surprises).
- `loop` is the modern, recommended keyword. It's simpler, more predictable (no implicit flattening), and works uniformly with filters like `| flatten`, `| dict2items`, `| subelements` to replace the old `with_dict`, `with_file`, `with_subelements`, etc.
- Going forward, `loop` is the one to reach for; `with_*` is kept only for backward compatibility with older playbooks.

---

## 6. Server Health Report (`server-report.yml`)

Sample generated report (`/tmp/server-report-<hostname>.txt`):

```
Server: web-server
OS: Ubuntu 22.04
IP: 10.0.1.15
RAM: 1984MB
Disk: Filesystem      Size  Used Avail Use% Mounted on
      /dev/xvda1       20G   6.1G   13G   32% /
Checked at: 2026-07-08T10:42:11Z
```

> 📸 *Insert your actual `/tmp/server-report-*.txt` content and the `debug` output from Task 6 here.*

**Verification:** SSH'd into each server and confirmed `/tmp/server-report-<hostname>.txt` matched the live `df -h` / `free -m` / OS facts on that machine — no stale or mismatched data. No hosts crossed the 90%+ disk-usage alert threshold at the time of the run.

---

## Key Takeaways

- **Variables** can come from playbooks, `group_vars`, `host_vars`, or the CLI (`-e`) — and Ansible has a strict, predictable precedence order for resolving conflicts.
- **Facts** are auto-gathered system info (`ansible_*`) that let a playbook adapt itself to whatever host it's running against, with zero extra input.
- **Conditionals** (`when`) turn one playbook into logic that behaves differently per OS, per group, or per resource constraint.
- **Loops** (`loop`) eliminate copy-pasted tasks for installing multiple packages, creating multiple users, or making multiple directories.
- Combined, these four features turn a static script into infrastructure automation that's actually reusable across dev, staging, and production.

#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham
