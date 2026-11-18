# Week 3: Base host role (hardening)

**Date:** 18 Nov 2026 · **Status:** done

## Goal

Turn the Week 2 bootstrap playbook into a reusable `base` role that hardens SSH, enables a firewall and keeps the OS patched.

## What I did

### Step 1: Scaffold the role

#### The role skeleton

`ansible-galaxy role init --init-path roles base` created `roles/base/` with the standard layout. What each folder is for, and whether I kept it:

| Folder | What it's for | Kept? |
| --- | --- | --- |
| `defaults/` | The role's default variable values. Lowest precedence of almost any variable source, so `group_vars`, `host_vars`, play vars and `-e` all override them. | Yes |
| `vars/` | Role variables with high precedence. They beat inventory `group_vars` and `host_vars`, so they are hard to override. | No |
| `tasks/` | The work the role does. `main.yml` is the entry point and imports the smaller task files. | Yes |
| `handlers/` | Tasks that run only when another task notifies them and that task reported `changed`. They run once, at the end of the play. | Yes |
| `templates/` | Jinja2 files. Ansible fills in variables and logic, then copies the result to the server. | Yes |
| `files/` | Static files copied exactly as they are, with no variables. | No |
| `meta/` | Role metadata: author, description, license, supported platforms, minimum Ansible version and role dependencies. | Yes |
| `tests/` | A sample inventory and playbook for testing the role on its own, usually in CI. | No |

Why I deleted three of them:

- **`vars/`:** nothing in this role needs high precedence. Worse, a variable put there by mistake can't be overridden from `group_vars`, which is exactly what I want to be able to do.
- **`files/`:** everything this week is templated, so there are no static files yet.
- **`tests/`:** `site.yml` run against the test VM already does this job.

`meta/main.yml` was filled in with real values in place of the generated placeholders: the role's description, license MIT (matching the repo's `LICENSE`), `min_ansible_version: "2.21"` and Ubuntu as the platform. The version is quoted because YAML would otherwise read it as a number, and as a number `2.10` turns into `2.1`.

#### One entry point: `playbooks/site.yml`

One play, `Apply base role to homelab servers`, targets the `homelab` group with `become: true` and applies the `base` role. `site.yml` is the only playbook I run from now on, and `bootstrap.yml` is removed (`git rm`).

#### Week 2 tasks, split into small files

`roles/base/tasks/main.yml` holds no tasks of its own. It imports three files with `ansible.builtin.import_tasks`, in this order:

| File | Task | Why it's separate |
| --- | --- | --- |
| `updates.yml` | Update the apt cache and upgrade all packages | Patching. Unattended upgrades join it in Step 4. |
| `packages.yml` | Install `base_packages` | Runs after the upgrade, so packages install from a fresh index. |
| `system.yml` | Set the timezone | Settings about the machine itself. The hostname joins it in Step 4. |

A task file is just a list of tasks: no `hosts`, no `become` and no `tasks:` key. The play in `site.yml` supplies those.

The task names in `main.yml` never appear in the run output. `import_tasks` is **static**: Ansible copies the imported tasks into place when it reads the playbook, so the import line itself never runs as a task. Its sibling `include_tasks` is dynamic and does show up. The difference matters for tags in Step 5.

#### Defaults and precedence

`roles/base/defaults/main.yml` now holds the role's two variables:

- `base_packages`, moved out of `group_vars` (git, curl, wget, vim, htop, dnsutils).
- `base_timezone: Etc/UTC`, a neutral default.

`inventories/test/group_vars/all/vars.yml` overrides `base_timezone` with `America/New_York`. `ansible_python_interpreter` stays there, because it's a connection setting for the inventory, not a role setting. The timezone task uses the variable in its module argument and at the end of its name (`Set timezone to {{ base_timezone }}`), so the output always shows the value actually being applied.

To prove the override works in both directions:

1. With the override in place, a normal run shows `Set timezone to America/New_York` and `changed=0`.
2. With the override commented out, `--check` shows `Set timezone to Etc/UTC` and `changed=1`: the default takes over.
3. With the override restored, a normal run is back to `changed=0`.

### Step 2: Harden SSH

Before touching SSH I took the snapshot `02-before-ssh` and kept a second SSH session open. If a bad config had locked me out, I could still fix it from the open session or roll back to the snapshot.

#### What was already there

`/etc/ssh/sshd_config.d/` held one file, `50-cloud-init.conf` (owner root, mode `600`), written by the Ubuntu installer. Its only line is `PasswordAuthentication yes`. That's why password login still worked after Week 1.

`sshd_config` includes every `sshd_config.d/*.conf` file near its top, in alphabetical order, and for most settings sshd keeps the **first** value it reads. So my file is named `10-hardening.conf`: it's read before `50-cloud-init.conf`, and its values win. If I had named it `60-` or `99-`, the cloud-init `yes` would have won and my `no` would have been ignored without any error.

I left `50-cloud-init.conf` alone on purpose. My `10-` file wins whatever that file says, so I don't need to delete it, and the hardening doesn't depend on cloud-init never rewriting it.

#### The files

| File | What it does |
| --- | --- |
| `tasks/ssh.yml` | One task, `Configure SSH hardening`. It renders the template to `/etc/ssh/sshd_config.d/10-hardening.conf` with owner and group root and mode `0600` (the same as the cloud-init file), and notifies `Restart SSH`. |
| `templates/sshd_hardening.conf.j2` | A `# {{ ansible_managed }}` header, then one sshd setting per line, each filled from a variable. |
| `handlers/main.yml` | `Restart SSH` restarts the `ssh` service. It runs only when the file actually changed. |
| `defaults/main.yml` | Five new `base_ssh_*` variables (below), so any of them can be overridden from `group_vars`. |
| `tasks/main.yml` | Imports `ssh.yml` last. If the SSH change ever fails, the updates, packages and timezone have already been applied. |

The settings:

| sshd setting | Variable | Value | Why |
| --- | --- | --- | --- |
| `PasswordAuthentication` | `base_ssh_password_authentication` | `no` | The point of this step: keys only. |
| `KbdInteractiveAuthentication` | `base_ssh_kbd_interactive_authentication` | `no` | The other way sshd can ask for a password. Ubuntu's main config already says `no`; stating it here means my hardening doesn't rely on that default. |
| `PermitRootLogin` | `base_ssh_permit_root_login` | `no` | The default, `prohibit-password`, still lets root log in with a key. I log in as `test-homelab` and use sudo. |
| `PubkeyAuthentication` | `base_ssh_pubkey_authentication` | `yes` | Already the default. Written down so the file documents the only way in. |
| `MaxAuthTries` | `base_ssh_max_auth_tries` | `3` | The default is 6. Fewer guesses per connection. |

The yes/no values are quoted strings (`"no"`, `"yes"`). Unquoted, YAML reads `no` as the boolean `false`, the template would write `False`, and sshd would reject it.

`AllowUsers` was left out for now. One wrong username there locks everyone out, and the settings above already close password login.

#### The safety net: `validate`

`validate: /usr/sbin/sshd -t -f %s` makes Ansible render the file to a temporary path, run sshd's own config test on it, and move it into place only if the test passes. A typo in a setting name fails the task on my side instead of reaching the server and locking me out.

`validate` only runs on a real run. In `--check` mode the module stops before copying, so the test never happens (checked in the `copy` module's source, which `template` uses). A preview can't prove the config is valid; only a real run on the test VM can.

The `ansible_managed` header tells anyone who opens the file on the server that Ansible owns it and will overwrite manual edits on the next run.

#### Restart the service, not the socket

Ubuntu 26.04 starts SSH through `ssh.socket`. For these settings, restarting the `ssh` service is enough, and sessions that are already open stay up. The `daemon-reload` plus `ssh.socket` restart is only needed when the port changes.

In `--check` mode the handler still shows up as `changed`, but it only reports what it would do. SSH isn't actually restarted.

#### Proving it

1. **Baseline:** before any change, `sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractive|permitrootlogin|pubkeyauthentication|maxauthtries'` showed `passwordauthentication yes`. `sshd -T` prints the settings sshd actually uses after reading every file, which is what matters, not what one file says.
2. **Preview:** `ansible-playbook playbooks/site.yml -K --check --diff` showed the exact file Ansible would write (see Evidence).
3. **Apply:** the real run reported `changed=2` (the template, then the handler). A second run reported `changed=0`.
4. **After:** `sshd -T` showed all five hardened values.
5. **Password login refused:** `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password test-homelab` from WSL was rejected with `Permission denied (publickey)`. The server no longer even offers a password prompt.
6. **Key login still works:** a new `ssh test-homelab` session logged in with my key.

`-K` still asks for my sudo password. Turning off SSH passwords doesn't affect `become`, because that's sudo's password, not SSH's.

### Step 3: Firewall with UFW

Before starting I applied the `PubkeyAuthentication` spelling fix with a real run, then took the snapshot `03-after-ssh`, so the snapshot matches the code.

#### The files

| File | What changed |
| --- | --- |
| `defaults/main.yml` | `ufw` added to `base_packages`, and a new variable `base_ufw_allowed: []` (extra ports to open, empty for now). |
| `tasks/firewall.yml` | Four new tasks, all `community.general.ufw` (below). |
| `tasks/main.yml` | Imports `firewall.yml` after `ssh.yml`. |

Adding `ufw` to `base_packages` changed nothing: Ubuntu Server already ships it. It's listed anyway so the role doesn't rely on that default, the same reason `KbdInteractiveAuthentication` is spelled out in Step 2.

#### Task order

UFW does nothing while it's inactive, so rules and policies can be set safely first. The moment it's switched on is the risky one, so that task goes last.

| # | Task | What it does |
| --- | --- | --- |
| 1 | Allow SSH through the firewall | `rule: limit` on `22/tcp`, comment `SSH`. Allows SSH but blocks an address that opens 6 or more connections in 30 seconds. |
| 2 | Allow additional ports through the firewall | Loops over `base_ufw_allowed`. `proto` defaults to `tcp`; `comment` uses `default(omit)`, so it's left out entirely when an entry has none. The list is empty, so the task shows `skipping`. |
| 3 | Set default firewall policies | Loops over two items: deny incoming, allow outgoing. |
| 4 | Enable the firewall | `state: enabled`. The only task that can lock me out, and by the time it runs the SSH rule already exists. |

#### Design decisions

- **SSH is not in `base_ufw_allowed`.** If a later week overrides the list in `group_vars` (Pi-hole's ports in Week 6) and forgets to copy the SSH entry, the next run would lock me out. A separate task means SSH is always allowed, whatever the list says.
- **Port 22 is written as a literal, not a variable.** The sshd template has no `Port` line, so a `base_ssh_port` variable would only move the firewall rule while sshd kept listening on 22: a lockout waiting to happen. If the port ever changes, the template and the firewall will read the same variable.
- **The default policies are stated, not assumed.** UFW's built-in defaults are already deny incoming and allow outgoing, but the role says so explicitly, so the firewall's behaviour is visible in the code.

#### Check mode overestimated the changes

The `--check` preview reported `changed=3`; the real run reported `changed=2`. The difference is the policy task: `changed` in check mode, `ok` for real.

While UFW is inactive, `ufw status verbose` prints only `Status: inactive`, with no `Default:` line. In check mode the module reads that output to decide whether the policy is already set, finds nothing, and assumes a change. In a real run it runs `ufw default deny incoming`, compares the state before and after, and sees that nothing changed, because the values already matched UFW's built-in defaults. Check mode is a preview, not a promise.

#### Proving it

1. **Baseline:** `sudo ufw status verbose` on the VM showed `Status: inactive`.
2. **Preview:** `--check --diff` showed the SSH rule, both policies and the enable as changes, and the extra-ports task as `skipping`.
3. **Apply:** with a second SSH session open, the real run reported `changed=2`. The second run reported `changed=0`.
4. **After:** `ufw status verbose` shows the firewall active, deny incoming and allow outgoing, and `22/tcp` as `LIMIT IN` for both IPv4 and IPv6 (see Evidence).
5. **Blocked ports are dropped:** from WSL, `nc -zv -w 3 192.168.77.50 80` timed out. Nothing listens on port 80, so without the firewall the answer would have been an immediate `Connection refused`. A timeout means UFW dropped the packet before the VM could answer.
6. **SSH still works:** a new `ssh test-homelab` session logged in with my key.

### Step 4: Patching and identity

#### Automatic updates: what was already there

Same rule as Step 2: look before writing. `/etc/apt/apt.conf.d/` already held `20auto-upgrades`, written by the installer, with both settings on:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

The folder also held `10periodic`, which sets `Update-Package-Lists "0"`. Yet `apt-config dump` reported `"1"`. apt reads these files in alphabetical order and the **last** value wins, so `20auto-upgrades` overrides `10periodic`. That's the opposite of sshd in Step 2, where the first value wins. Two config systems on the same server, two opposite rules: the file-name prefix has to be chosen for the program reading it.

`Automatic-Reboot` wasn't set anywhere (it's commented out in `50unattended-upgrades`), so unattended-upgrades used its built-in default, `false`.

#### The files

| File | What it does |
| --- | --- |
| `tasks/updates.yml` | Three new tasks after the upgrade: install `unattended-upgrades`, then render the two templates below (owner and group root, mode `0644`, the same as the installer's file). |
| `templates/20auto-upgrades.j2` | Replaces the installer's file under the same name, so Ansible and `dpkg-reconfigure unattended-upgrades` work on one file. Refreshes the package lists daily, runs unattended upgrades daily, and adds `AutocleanInterval "7"` to clear the downloaded package cache weekly (it was `0`, never). |
| `templates/52unattended-upgrades-local.j2` | The reboot settings. Named `52` so it's read **after** `50unattended-upgrades` and its values win. |
| `defaults/main.yml` | Five new `base_auto_upgrades_*` variables. |

Both templates start with `// {{ ansible_managed }}`: apt config files use `//` for comments, not `#`. No handler is needed: the apt timers read these files fresh every time they run, so there's nothing to restart.

`unattended-upgrades` is installed in `updates.yml` rather than added to `base_packages`, because `updates.yml` runs before `packages.yml` and this keeps all the patching in one file.

Two variable details:

- **`base_auto_upgrades_reboot: true`** is a real boolean. The template writes it with `{{ base_auto_upgrades_reboot | lower }}`, because Jinja would otherwise print `True` and apt expects `true`.
- **`base_auto_upgrades_reboot_time: "04:00"`** must be quoted. Ansible reads YAML 1.1, where an unquoted `04:00` is a base-60 number and becomes `240`. The same family of trap as `no` turning into `False` in Step 2.

#### Decision: automatic reboots on, at 04:00

A server that applies a kernel or library fix isn't protected until it reboots, so automatic reboots are on. The cost is that from Week 10 this server is the home's only DNS and DHCP server, and a reboot is a short outage for everyone.

The timing matters. `apt-daily-upgrade.timer` runs unattended upgrades between about 6:00 and 7:00 a.m. With the default reboot time (`now`), the server would reboot right after upgrading, while the household is waking up. With `Automatic-Reboot-Time "04:00"`, unattended-upgrades schedules the reboot for 04:00, and because that time has already passed when the upgrade runs, the reboot happens at 04:00 the **next** morning. The trade-off: up to about a day before a fix that needs a reboot takes effect, in exchange for the outage landing when nobody is using the network. If the server doesn't come back after a reboot, the Pi-hole-down fallback runbook (turn DHCP back on at the BGW320) still applies.

#### Proving it

1. **Preview:** `--check --diff` showed `changed=2`. The `20auto-upgrades` diff added only the `ansible_managed` line and `AutocleanInterval "7"`: the two existing lines were untouched, which proves the template reproduces the installer's file. `52unattended-upgrades-local` showed as a new file, with lowercase `"true"` and `"04:00"` intact.
2. **Apply:** the real run reported `changed=2`. The second run reported `changed=0`.
3. **After:** `apt-config dump` shows all the new values (see Evidence).
4. **Dry run:** `sudo unattended-upgrade --dry-run --debug` started without errors. Its allowed origins are `resolute` and `resolute-security` plus the two ESM security sources: by default, only security updates are applied automatically, not everything in `resolute-updates`.
5. **Timers:** `systemctl list-timers 'apt-daily*'` shows the next upgrade run at 06:27, inside the 6:00–7:00 window the 04:00 reasoning depends on.

The same preview also confirmed the Step 3 explanation: with UFW now active, `Set default firewall policies` reports `ok` in check mode, because `ufw status verbose` finally prints a `Default:` line the module can read.

#### Hostname

`tasks/system.yml` gained one task, `Set hostname to {{ inventory_hostname }}`, using `ansible.builtin.hostname`. The server's name now comes from the inventory, so the inventory is the single place a server is named.

The VM was already called `test-homelab`, so a normal run would only ever show `ok` and prove nothing. To make the task do real work, I broke the name first with `sudo hostnamectl set-hostname wrongname`.

I expected `sudo` to warn `unable to resolve host wrongname`, because `/etc/hosts` still maps `127.0.1.1` to `test-homelab` and `ansible.builtin.hostname` never edits `/etc/hosts`. There was no warning. The likely reason: Ubuntu 26.04 sends name lookups to `systemd-resolved`, which always answers for the machine's current hostname, so the lookup succeeded anyway. (sudo-rs may also skip the lookup entirely.) Predicted one thing, got another; the lesson is to test the prediction rather than trust it.

1. **Preview:** `--check --diff` showed `-hostname = wrongname` / `+hostname = test-homelab` and `changed=1`.
2. **Apply:** the real run reported `changed=1`; the second run `changed=0`.
3. **After:** `hostnamectl` shows `Static hostname: test-homelab`.

### Step 5: Tags

Every `import_tasks` line in `tasks/main.yml` got one tag: `updates`, `packages`, `system`, `ssh` and `firewall`. That's the guide's four plus `system`, for the timezone and hostname.

One tag per import line is enough because `import_tasks` is static (Step 1): Ansible copies the imported tasks into place when it reads the playbook, and a tag on the import line is copied onto every task inside. No task file needed editing.

| Command | What it did |
| --- | --- |
| `--list-tags` | Listed `[firewall, packages, ssh, system, updates]` without connecting to the VM. |
| `--list-tasks --tags firewall` | Listed exactly the four firewall tasks, each with `TAGS: [firewall]`. |
| `--tags firewall` | Ran Gathering Facts plus the firewall tasks: `ok=4 changed=0 skipped=1`. |
| `--skip-tags updates` | Ran everything except the four update tasks: `ok=8 changed=0 skipped=1`, against `ok=12` for a full run. |

Gathering Facts still runs with `--tags`, because Ansible gives it the special tag `always`. `--list-tags` and `--list-tasks` are worth running first on a long playbook: they show what a tag selects without touching the server.

### Step 6: Security baseline

[`docs/security-baseline.md`](../security-baseline.md) lists the 12 controls the role applies. For each one: how it's implemented, how to verify it on a running server, and the Security+ concept it maps to. It also records the decisions and accepted risks from this week:

- the cloud-init SSH file left in place
- no `AllowUsers` yet
- **no passwordless sudo:** a stolen SSH key alone shouldn't give root, so `-K` stays until Ansible Vault in Week 4
- automatic reboots at 04:00

It ends with the known gaps (Docker bypassing UFW, a flat network) and the controls planned for later weeks.

## Checks

- [x] Password login refused: `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password test-homelab` → `Permission denied (publickey)`
- [x] Root login refused: `ssh root@192.168.77.50` → `Permission denied (publickey)`
- [x] `sudo ufw status verbose` shows default deny incoming and only `22/tcp` allowed (rate-limited, IPv4 and IPv6)
- [x] Second run `changed=0`, and the `Restart SSH` handler doesn't run
- [x] `--tags firewall` runs only the firewall tasks
- [x] Rebuild from zero: after restoring `01-fresh-install`, one run of `site.yml` reported `changed=8` (upgrade, both apt files, timezone, the SSH file and its restart, the SSH rule, enabling UFW) and a second run `changed=0`
- [x] `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings, `production` profile

On the fresh VM, `Install required packages`, `Install unattended-upgrades` and `Set hostname` all reported `ok`: Ubuntu Server already ships every package in the list, and the installer had already named the VM `test-homelab`. They're in the role anyway, so a server that doesn't match those defaults still ends up in the same state.

No lockouts this week. The snapshots and the second open SSH session were never needed, which is the point of having them.

## What broke and how I fixed it

1. **`yaml[comments]: Missing starting space in comment`** in four files. `ansible-galaxy role init` writes `#SPDX-License-Identifier: MIT-0` at the top of every `main.yml`, with no space after `#`. I added the space and made the license match the repo's MIT.
2. **`Empty playbook, nothing to do`.** `site.yml` hadn't been saved yet, the same mistake as Week 2. Auto save is now on in VS Code.
3. **`The role 'base' was not found`.** Ansible looks for roles in a `roles/` folder next to the playbook, in the default `roles_path` and in the playbook's own folder. My playbook is in `playbooks/` and the role is in the repo-root `roles/`, so none of those matched. Adding `roles_path` to `[defaults]` in `ansible.cfg` fixed it. A relative path there is resolved from the folder `ansible.cfg` is in, not from my current directory.
4. **`No route to host`.** The VM wasn't running. Nothing answered at `192.168.77.50`, which is different from SSH refusing the connection.
5. **The role ran, but only "Gathering Facts" appeared.** I had deleted `tasks/main.yml` while moving the tasks into their own files. A role with no `tasks/main.yml` is still valid, so Ansible and `ansible-lint` both stayed silent. `main.yml` is the only task file Ansible loads on its own; everything else must be imported from it.
6. **`import_tasks` written as a list** (`- updates.yml` under the module). Caught in review before running. One file is a plain value, not a list item.
7. **A precedence test that couldn't fail.** The first run after adding the override showed `changed=0` and looked like a pass, but the task still had `America/New_York` hard-coded and `group_vars` set `timezone:` instead of `base_timezone:`. Neither variable was read anywhere. After making the task use `base_timezone` and fixing the name, the `--check` test above proved the override really works.
8. **`base_packages` in two places.** It was still in `group_vars` after I added it to `defaults/`. Identical lists hide the problem until one copy is edited. It now lives only in `defaults/`.
9. **Variables named after the sshd settings.** My first `defaults/main.yml` used keys like `KbdInteractiveAuthentication: "no"` while the template looked up `base_ssh_kbd_interactive_authentication`. VS Code showed 4 problems from ansible-lint's `var-naming` rule: role variables must be lowercase and start with the role name. I renamed them all to `base_ssh_*`.
10. **`'base_ssh_public_key_authentication' is undefined`.** After renaming the defaults, the template still used the old name for one variable. The first `--check` run stopped at the SSH task before anything on the server changed. The template and defaults now use the same names.
11. **`PublicKeyAuthentication` is not an sshd setting.** The real name is `PubkeyAuthentication`. Caught in review; if it hadn't been, `validate` would have rejected the file with `Bad configuration option`. My next version wrote `PubKeyAuthentication` with a capital K. That works, because sshd ignores capitalization in setting names, but the template now uses the official spelling from the man page.
12. **`ansible.builtin.ufw` doesn't exist.** My first `firewall.yml` used it for every task, and VS Code underlined every parameter. UFW isn't part of ansible-core; the module is `community.general.ufw`, from the collection pinned in `requirements.yml`. Because Ansible couldn't find the module, it couldn't check any parameter either, so one wrong name produced most of the 7 problems.
13. **Caught in review before running:**
    - An `Install ufw` task with `ansible.builtin.apt`, duplicating `ufw` in `base_packages` (the same kind of duplication as mistake 8). Its `update_cache: yes` also tripped ansible-lint's `truthy` rule, which wants `true`.
    - The SSH rule used `rule: allow` with `name: OpenSSH` (UFW's built-in profile). The step calls for `rule: limit`, and I switched to `port: "22"`/`proto: tcp`, so `ufw status` shows the exact port and the rule doesn't depend on a profile file shipped by another package.
    - The enable task had `enabled: yes` next to `state: enabled`. `enabled` isn't a ufw parameter.
    - There was no task for the default policies.
14. **`one of the following is required: state, default, rule, logging`.** The first `--check` run failed on `Set default firewall policies`, both loop items, even though the task was meant to set `default`. I had misspelled the `default` key, so the module never received it. The message is misleading: the module checks for a missing command before it reports an unknown parameter, so a misspelled key shows up as "required" rather than "unsupported".
15. **Two exact-name mismatches in Step 4.**
    - `Could not find or access '20auto-upgrades.j2'`: the file was saved as `20auto_upgrades.j2`, with an underscore. The error lists every path Ansible searched; the first one, `roles/base/templates/`, was the right folder, so the problem had to be the name.
    - `'base_auto_upgrades_unattended_upgrade' is undefined`: `defaults/main.yml` had `base_auto_upgrades_unattended_upgrades`, with an extra `s`. The same mistake as number 10. `grep -rnoE 'base_auto_upgrades_[a-z_]+' roles/base/ | sort -t: -k3` lists every variable name next to its partner, so the one without a match stands out.

## Evidence

### Step 1

From zero, after restoring `01-fresh-install`, then a second run:

```text
test-homelab : ok=4  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
test-homelab : ok=4  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

Override test (`--check` with the `group_vars` override commented out, then a normal run with it restored):

```text
TASK [base : Set timezone to Etc/UTC]
changed: [test-homelab]
test-homelab : ok=4  changed=1  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0

TASK [base : Set timezone to America/New_York]
ok: [test-homelab]
test-homelab : ok=4  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 8 files, `production` profile.

### Step 2

The `--check --diff` preview of the new file (this run still had the capital K in `PubKeyAuthentication`, fixed afterwards):

```text
TASK [base : Configure SSH hardening]
--- before
+++ after: .../sshd_hardening.conf.j2
@@ -0,0 +1,11 @@
+# Ansible managed
+
+PasswordAuthentication no
+
+KbdInteractiveAuthentication no
+
+PermitRootLogin no
+
+PubKeyAuthentication yes
+
+MaxAuthTries 3

changed: [test-homelab]

RUNNING HANDLER [base : Restart SSH]
changed: [test-homelab]
```

- Real run `changed=2`, second run `changed=0`.
- `sshd -T` after the run shows all five hardened values; forced password login refused with `Permission denied (publickey)`; key login works.
- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 9 files, `production` profile. Python also printed `SyntaxWarning: invalid decimal literal` while ansible-lint loaded. It's a Python warning, not a lint finding, and the pass result stands. (`PYTHONWARNINGS=error::SyntaxWarning ansible-lint playbooks/site.yml` turns it into a traceback that names the file it came from.)

### Step 3

`sudo ufw status verbose` on the VM after the real run:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT IN    Anywhere                   # SSH
22/tcp (v6)                LIMIT IN    Anywhere (v6)              # SSH
```

Real run, then second run:

```text
test-homelab : ok=8  changed=2  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=8  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

A blocked port times out instead of being refused:

```text
$ nc -zv -w 3 192.168.77.50 80
nc: connect to 192.168.77.50 port 80 (tcp) timed out: Operation now in progress
```

- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 10 files, `production` profile.

### Step 4

`apt-config dump | grep -E 'Periodic|Automatic-Reboot'` on the VM after the run:

```text
APT::Periodic "";
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "0";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

`sudo unattended-upgrade --dry-run --debug`, the allowed origins:

```text
Allowed origins are: o=Ubuntu,a=resolute, o=Ubuntu,a=resolute-security, o=UbuntuESMApps,a=resolute-apps-security, o=UbuntuESM,a=resolute-infra-security
```

`systemctl list-timers 'apt-daily*'`:

```text
NEXT                        LEFT LAST                              PASSED       UNIT
Wed 2026-11-18 19:48:19 EST   6h Wed 2026-11-18 11:30:58 EST 1h 43min ago apt-daily.timer
Thu 2026-11-19 06:27:35 EST  17h Wed 2026-11-18 11:29:42 EST 1h 45min ago apt-daily-upgrade.timer
```

- Real run `changed=2`, second run `ok=11  changed=0  skipped=1`.
- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 10 files, `production` profile.

Hostname: the `--check --diff` preview after renaming the VM to `wrongname`, then the runs:

```text
TASK [base : Set hostname to test-homelab]
--- before
+++ after
@@ -1 +1 @@
-hostname = wrongname
+hostname = test-homelab

changed: [test-homelab]

test-homelab : ok=12  changed=1  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=12  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

### Step 5

```text
$ ansible-playbook playbooks/site.yml --list-tasks --tags firewall
  play #1 (homelab): Apply base role to homelab servers TAGS: []
    tasks:
      base : Allow SSH through the firewall     TAGS: [firewall]
      base : Allow additional ports through the firewall        TAGS: [firewall]
      base : Set default firewall policies      TAGS: [firewall]
      base : Enable the firewall        TAGS: [firewall]
```

- `--tags firewall`: `ok=4 changed=0 skipped=1`. `--skip-tags updates`: `ok=8 changed=0 skipped=1`.
- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 10 files, `production` profile.

### Rebuild from zero

After restoring `01-fresh-install`, the first run, then the second:

```text
TASK [base : Update apt cache and upgrade all packages]   changed
TASK [base : Install unattended-upgrades]                 ok
TASK [base : Configure automatic updates]                 changed
TASK [base : Configure automatic reboots]                 changed
TASK [base : Install required packages]                   ok
TASK [base : Set timezone to America/New_York]            changed
TASK [base : Set hostname to test-homelab]                ok
TASK [base : Configure SSH hardening]                     changed
TASK [base : Allow SSH through the firewall]              changed
TASK [base : Allow additional ports through the firewall] skipping
TASK [base : Set default firewall policies]               ok (both items)
TASK [base : Enable the firewall]                         changed
RUNNING HANDLER [base : Restart SSH]                      changed

test-homelab : ok=13  changed=8  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=12  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

```text
$ ssh root@192.168.77.50
root@192.168.77.50: Permission denied (publickey).
```

## Notes for later weeks

- **`AllowUsers`:** skipped for now (recorded in `security-baseline.md`). Revisit once the user list is settled.
- **`-K` on every run (Week 4):** passwordless sudo was declined. Ansible Vault can hold the become password instead.
- **Docker skips UFW (Weeks 4–5).** Docker writes its own iptables rules, and ports it publishes for containers on bridge networks (Portainer, NPM) bypass UFW completely. Pi-hole on the host network does go through UFW, so Week 6 adds its ports (53 TCP/UDP, 67 UDP for DHCP, 8081 for the web UI) to `base_ufw_allowed`. Listed as a known gap in `security-baseline.md`; it needs its own control before Week 10.
- **Removing a port from `base_ufw_allowed` doesn't close it.** Ansible only adds UFW rules. Taking an entry out of the list leaves the rule on the server; closing it needs a task with `delete: true`.
- **SSH arrives from `192.168.77.100`, the Windows desktop.** WSL's traffic leaves through Windows' own address. If I ever limit SSH by source (`from_ip`), that's the address to allow, and it's a dynamic lease today, so it would need a DHCP reservation first.
- **"Running on the development release."** The `unattended-upgrade --dry-run --debug` output printed this on 26.04.1 LTS, which isn't a development release. The run carried on normally, so upgrades still work. Worth finding out why (possibly outdated release data on the VM) before production.
- **`/etc/hosts` isn't managed.** `ansible.builtin.hostname` doesn't touch it. On this VM the names already match. If production's inventory name differs from the name the ThinkPad was installed with, the `127.0.1.1` line will be stale; an `ansible.builtin.lineinfile` task can keep it in step.
