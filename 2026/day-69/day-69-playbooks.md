# Day 69 — Ansible Playbooks

Repo layout for today:

```
2026/day-69/
├── inventory.ini
├── install-nginx.yml
├── essential-modules.yml
├── nginx-config.yml
├── multi-play.yml
├── files/
│   ├── app.conf
│   └── nginx.conf
└── day-69-playbooks.md
```

All playbooks below target **Ubuntu/Debian** managed nodes, so they use the
`apt` module instead of `yum`. Nginx's default web root on Ubuntu is
`/var/www/html`, so the index-page tasks were adjusted from
`/usr/share/nginx/html` accordingly.

---

## Task 1 — My First Playbook (annotated)

`install-nginx.yml`:

```yaml
---
- name: Install and start Nginx on web servers   # PLAY: targets the "web" group
  hosts: web                                     # inventory group this play runs against
  become: true                                   # escalate to root (sudo) for every task below

  tasks:
    - name: Install Nginx                        # TASK 1
      apt:                                        # MODULE: apt (Ubuntu/Debian package manager)
        name: nginx
        state: present                            # install if not already installed
        update_cache: true                        # apt-get update before installing

    - name: Start and enable Nginx                # TASK 2
      service:                                    # MODULE: service
        name: nginx
        state: started                            # start it now if not running
        enabled: true                              # start on boot too

    - name: Create a custom index page             # TASK 3
      copy:                                        # MODULE: copy
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /var/www/html/index.html
```

Run it:

```bash
ansible-playbook -i inventory.ini install-nginx.yml
```

**First run** — every task reports `changed` (Nginx wasn't installed, wasn't
running, index page didn't exist).

**Second run** — every task reports `ok`, because the desired state already
matches reality. Nothing is reinstalled, nothing is restarted, the file isn't
rewritten. This is **idempotency**: playbooks describe a *desired end state*,
not a sequence of commands, so re-running them is always safe.

Verification: `curl http://<web-server-public-ip>` should return
`<h1>Deployed by Ansible - TerraWeek Server</h1>`.

*(Add your own terminal screenshots here: one showing the first run with
`changed` tasks, and one showing the second run with all `ok`.)*

---

## Task 2 — Playbook Structure, Q&A

**What is the difference between a play and a task?**
A **play** maps a group of hosts (from the inventory) to a set of work to do
on them — it's the outer block with `hosts:`, `become:`, and `tasks:`. A
**task** is a single unit of work inside a play — one call to one module
(install a package, copy a file, restart a service). A play is the "who and
under what settings"; a task is the "what, one step at a time."

**Can you have multiple plays in one playbook?**
Yes. A playbook is just a YAML list of plays. Each play can target a
different host group, run as a different user, and have entirely different
tasks — see `multi-play.yml` in Task 6, which has three separate plays for
`web`, `app`, and `db`.

**What does `become: true` do at the play level vs the task level?**
At the **play level**, it sets the default privilege-escalation behavior for
every task in that play (all tasks run as root/sudo unless overridden). At
the **task level**, it overrides that default for just that one task — e.g.
you could set `become: true` on the play but `become: false` on a single
task that intentionally needs to run as the normal login user.

**What happens if a task fails — do remaining tasks still run?**
By default, no. Ansible stops executing further tasks *for that host* the
moment a task fails, and marks that host as failed for the rest of the play
(other, unaffected hosts continue normally). You can change this behavior
with `ignore_errors: true` on a task, or with `any_errors_fatal`/
`max_fail_percentage` at the play level, or by using `block`/`rescue` for
structured error handling.

---

## Task 3 — Essential Modules

`essential-modules.yml` exercises all seven modules:

| Module | What it does |
|---|---|
| `apt` | Installs/removes packages (Debian/Ubuntu equivalent of `yum`). `state: present` installs, `state: absent` removes. |
| `service` | Starts, stops, restarts, or enables/disables a system service. |
| `copy` | Copies a file (or inline `content:`) from the control node to the managed node, with owner/group/mode set in the same task. |
| `file` | Manages files, directories, and permissions — here used to create a directory with a specific owner and mode. |
| `command` | Runs a single command directly (no shell). Safer, but no pipes, redirects, `&&`, or environment variable expansion. |
| `shell` | Runs a command through `/bin/sh`, so pipes (`|`), redirects (`>`), and chaining (`&&`) work. |
| `lineinfile` | Ensures one specific line exists (or is updated) in a file, without touching the rest of the file. |

`files/app.conf` is a small sample config used by the `copy` task.

**`command` vs `shell` — when to use each:**
`command` executes the program directly via `exec`, bypassing the shell
entirely. It's safer (no shell injection risk, no accidental globbing) and
should be your **default choice** for running a single binary with
arguments — e.g. `df -h`, `mkdir`, `systemctl status foo`.

`shell` runs the command string through `/bin/sh -c "..."`, so it supports
shell features: pipes (`ps aux | wc -l`), redirection (`>`, `>>`),
environment variable expansion, and chaining with `&&` or `;`. Use `shell`
**only** when you actually need one of those shell features — and prefer a
dedicated module (`copy`, `lineinfile`, `apt`, etc.) over either one whenever
a purpose-built module exists, since modules are idempotent by design while
`command`/`shell` are not (hence adding `changed_when: false` to read-only
commands like `df -h` above, so they don't falsely report "changed" every
run).

---

## Task 4 — Handlers

`nginx-config.yml` deploys `files/nginx.conf` and only restarts Nginx when
that file actually changes, via `notify: Restart Nginx` paired with a
handler of the same name under `handlers:`.

**Before/after comparison:**

| Run | `Deploy Nginx config` task | Handler `Restart Nginx` |
|---|---|---|
| 1st run | `changed` (file didn't exist / differed) | **fires** — Nginx restarts |
| 2nd run | `ok` (file content already matches) | **does not fire** — no restart |

Key mechanics:
- A handler only runs if a task that `notify`s it reports `changed`.
- Handlers run **once**, at the end of the play, even if multiple tasks
  notify the same handler multiple times.
- If no notifying task changes anything, the handler is skipped entirely —
  which is exactly what you want in production: no unnecessary service
  restarts (and no unnecessary downtime) on runs where nothing changed.

*(Add screenshots here: first run showing `changed` + handler running, and
second run showing `ok` + no handler execution.)*

---

## Task 5 — Dry Run, Diff, and Verbosity

```bash
# Preview changes without applying them
ansible-playbook -i inventory.ini install-nginx.yml --check

# Preview + show the actual line-by-line file differences
ansible-playbook -i inventory.ini nginx-config.yml --check --diff

# Increasing levels of output detail
ansible-playbook -i inventory.ini install-nginx.yml -v     # verbose: shows task results/details
ansible-playbook -i inventory.ini install-nginx.yml -vv    # more verbose: adds module args
ansible-playbook -i inventory.ini install-nginx.yml -vvv   # connection-level debugging (SSH, etc.)

# Limit execution to one host
ansible-playbook -i inventory.ini install-nginx.yml --limit web-server

# See what would run, without running it
ansible-playbook -i inventory.ini install-nginx.yml --list-hosts
ansible-playbook -i inventory.ini install-nginx.yml --list-tasks
```

**Why `--check --diff` is the most important flag combination for
production:**
`--check` runs the playbook in "dry run" mode — Ansible connects to the real
hosts and evaluates every task's condition, but makes no actual changes, so
you can see exactly which tasks *would* report `changed` before you commit
to anything. `--diff` layers on top of that by showing the literal
before/after content differences for any file-based module (`copy`,
`template`, `lineinfile`, etc.), so instead of just "this file would change"
you see *what specifically* would change, line by line. Together they let
you review the blast radius of a playbook against production servers with
zero risk — the closest thing Ansible has to `terraform plan` — which is why
it should be a habit before any run against hosts that matter.

---

## Task 6 — Multiple Plays in One Playbook

`multi-play.yml` has three plays, one per inventory group (`web`, `app`,
`db`), each with its own `become`/`tasks`. When you run it:

```bash
ansible-playbook -i inventory.ini multi-play.yml
```

Ansible executes the plays **in order**, and each play only touches hosts in
its `hosts:` group — so Nginx is installed only on `web`, the
`gcc`/`make` build tools and `/opt/app` directory only appear on `app`, and
`mysql-client` plus `/var/lib/appdata` only appear on `db`. Hosts not in a
play's target group are simply skipped for that play's tasks.

**Verification:** `ansible web -i inventory.ini -m command -a "dpkg -l nginx"`
should succeed only on the web group; running the equivalent check for
`mysql-client` should succeed only on the db group.

---

## Summary

| Concept | Takeaway |
|---|---|
| Idempotency | Re-running a playbook only changes what's actually out of sync |
| Play vs Task | Play = hosts + settings; Task = one module call |
| Handlers | Run once, only when notified, at the end of the play |
| `--check --diff` | Safe, read-only preview of exactly what would change |
| `command` vs `shell` | Use `command` by default; `shell` only for pipes/redirects/chaining |
| Multiple plays | One playbook can configure entirely different host groups differently |
