# Week 2: Ansible fundamentals

**Date:** 17 Nov 2026 · **Status:** done

## Goal

Understand how Ansible talks to a server, and write a first playbook that runs cleanly twice in a row.

## What I did

### Configuration and inventory

- `ansible.cfg` in the repo root sets the inventory (`inventories/test/hosts.yml`), `remote_user`, and `retry_files_enabled = False`. `ansible-config dump --only-changed` confirms Ansible reads it.
- `host_key_checking` stays on. Turning it off would let Ansible connect to any machine that answers on that IP without checking that it is really my server.
- `inventories/test/hosts.yml` is a YAML inventory with one group, `homelab`, holding one host, `test-homelab` (`ansible_host: 192.168.77.50`).
- `inventories/test/group_vars/all/vars.yml` holds the variables for every host in this inventory: the package list (`base_packages`) and `ansible_python_interpreter: /usr/bin/python3`. Pinning the interpreter removed the "discovered Python interpreter at /usr/bin/python3.14" warning and keeps working when Ubuntu moves to a newer Python.
- `requirements.yml` lists the one collection the playbook needs, `community.general`.

### Ad-hoc commands

- `ansible homelab -m ansible.builtin.ping` returns `pong`. It is not an ICMP ping: it proves SSH and Python both work.
- `ansible homelab -m ansible.builtin.setup` prints the facts Ansible gathers about the host.
- Installing `htop` with `-b -K` and then running the same command again reports `ok` instead of `changed`: my first look at idempotency.

### The bootstrap playbook (`playbooks/bootstrap.yml`)

One play, `Bootstrap homelab servers`, targets the `homelab` group with `become: true` and runs three tasks:

| Task | Module | Notes |
| --- | --- | --- |
| Update apt cache and upgrade all packages | `ansible.builtin.apt` | `cache_valid_time: 3600` skips the index download if it is less than an hour old; `upgrade: dist` |
| Install required packages | `ansible.builtin.apt` | `name: "{{ base_packages }}"`: git, curl, wget, vim, htop, dnsutils |
| Set timezone to America/New_York | `community.general.timezone` | The server clock was on UTC |

Choices I made:

- **`upgrade: dist` over `safe`.** `dist` can add or remove packages to finish an upgrade, which Ubuntu kernel updates often need. On a server that only Ansible manages, that is what I want.
- **The package list goes straight to `name:` instead of a loop.** The guide said "loop over it", but passing the whole list installs everything in one apt transaction instead of six, and it is the form the apt module documentation recommends.
- **`dnsutils` is in the list** because it provides `dig`, which I need from Week 6 on.
- **Dropped `net-tools` and `python3-pip`.** `net-tools` (`ifconfig`, `netstat`) is deprecated; `ip` and `ss` replace it. Ubuntu blocks system-wide `pip install`, and the Docker library comes from apt (`python3-docker`) in Week 4.

### Running it

1. `--syntax-check`, then `ansible-lint` (clean on the `production` profile).
2. A dry run with `-K --check --diff`: 46 packages to upgrade (5 more held back by Ubuntu's phased updates) and the timezone change from `Etc/UTC` to `America/New_York`.
3. A real run, then a second run, which showed `changed=0`.
4. Moved the package list into `group_vars` and ran again.
5. Restored the `01-fresh-install` snapshot and ran from zero, twice.

### sudo-rs and `become`

Week 1 found that the VM uses sudo-rs, which is known to make Ansible's `become` hang. It didn't hang. The VM still runs sudo-rs `0.2.13-0ubuntu1.2`, and ansible-core 2.21.6 handles its password prompt. I kept sudo-rs and skipped the planned switch to classic sudo. The reasoning is in [ADR 0001](../adr/0001-keep-sudo-rs.md).

## What broke and how I fixed it

1. **`Empty playbook, nothing to do`.** The file wasn't saved. I now save before every run.
2. **Two plays instead of one.** I put a `-` in front of `hosts:`, which started a second play holding only `hosts`. A play's `name`, `hosts`, `become` and `tasks` all belong under one dash.
3. **ansible-lint couldn't find `community.general.timezone`** (`syntax-check[unknown-module]`), although `--syntax-check` passed. The `ansible` pipx package bundles the collection (13.5.1), but ansible-lint runs in its own pipx environment and can't see it. `ansible-galaxy collection install -r requirements.yml` first said "Nothing to do" because it found the bundled copy; `--force` installed 14.0.1 into `~/.ansible/collections`, which both tools read, and which Ansible now uses first.
4. **`yaml[new-line-at-end-of-file]`.** Fixed. VS Code's "Insert Final Newline" setting prevents it.
5. **`'base_packages' is undefined`** on the first from-zero run. I had created `inventories/group_vars/` instead of `inventories/test/group_vars/`. Ansible only reads `group_vars` next to the inventory file, so neither variable loaded (the interpreter warning was the other clue). `find . -name vars.yml` showed where the file was, and `ansible-inventory --host test-homelab` confirmed the fix. I then restored the snapshot again so the from-zero test really started from zero.
6. **A typo, `ansible-galazy`.** Ubuntu offered to install `ansible-core` 2.14 from apt to "fix" it. I didn't accept: a second, older Ansible on the PATH is the problem I removed in Week 1.

## Breaking it on purpose

| What I broke | What Ansible said | What it means |
| --- | --- | --- |
| Module name `ansible.builtin.aptt` | `couldn't resolve module/action 'ansible.builtin.aptt'` | Misspelled module or missing collection. Caught by `--syntax-check` before anything touches the server. |
| `state: present` moved two spaces left | `conflicting action statements: ansible.builtin.apt, state` | `state` became a task-level key, so Ansible thought the task named two modules. This error almost always means bad indentation. |
| `ansible_host` set to unused `192.168.77.253` | `UNREACHABLE! ... No route to host` | Ansible never got in. Different from `FAILED`, where it connected but a task went wrong. |
| Variable file in the wrong folder | `'base_packages' is undefined` | Ansible points to the exact line and column that used the variable. |

The wrong-IP test first "passed" because I hadn't saved `hosts.yml`. An unsaved file, or an SSH config alias with the same name as the inventory host, can hide a broken inventory. `ansible-inventory --host test-homelab` and `ansible homelab -m ansible.builtin.ping -vvv | grep ESTABLISH` show what Ansible really uses.

SSH connection errors, and what each one points to:

| Error | Meaning |
| --- | --- |
| `No route to host` | Nothing answered at that address. On the local subnet, no device has that IP. |
| `Connection timed out` | Packets are being dropped on the way, usually by a firewall. |
| `Connection refused` | The host is up, but nothing listens on that port (SSH stopped or moved to another port). |

## Questions

**What's the difference between the `command` and `shell` modules, and why avoid both?**
`command` starts the program directly with no shell in between, so pipes, redirects, `&&`, wildcards and `$VARIABLES` don't work. `shell` runs the line through `/bin/sh`, just like typing in a terminal, which also makes it open to injection if a variable ends up in the command. Avoid both when a module exists: Ansible can't tell what the command did, so it reports `changed` on every run, which breaks idempotency reporting, and `--check` can't predict the result.

**What is a fact? Name three you could use in a playbook.**
A fact is information Ansible collects about a server at the start of every play (the "Gathering Facts" task). Three examples:

- `ansible_distribution`: the OS name, for example to run a task only on Ubuntu.
- `ansible_default_ipv4.address`: the server's IP, to write into a config file without hard-coding it.
- `ansible_memtotal_mb`: total RAM in MB, to size a service or stop early on a host that is too small.

**Why is `apt` with `state: present` idempotent, while `command: apt-get install htop` isn't (in how Ansible reports it)?**
`apt` with `state: present` checks the system first. It reports `ok` when the package is already installed and `changed` only when it actually installs it. `command: apt-get install htop` runs blindly: Ansible doesn't read the output, so it reports `changed` on every run, even when htop is already there. Both leave htop installed; only the first tells the truth about it.

## Evidence

First run, then a second run against the same VM:

```text
test-homelab : ok=4  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
test-homelab : ok=4  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

From zero, after restoring `01-fresh-install` and moving the package list into `group_vars`:

```text
test-homelab : ok=4  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
test-homelab : ok=4  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `ansible homelab -m ansible.builtin.ping` returns `pong`.
- `ansible-lint playbooks/`: 0 failures, 0 warnings, `production` profile.

## Notes for later weeks

- **Week 3:** consider passwordless sudo for the Ansible user (no `-K` on every run, and no password prompt for sudo-rs to get wrong). `bootstrap.yml` splits into the base role's task files. Add `autoremove: true` to clean up old kernels.
- **Week 4:** the become password moves into Ansible Vault.
- **Phased updates:** 5 packages (`fwupd`, `snapd` and others) were held back. A later run will show them as `changed`; that's Ubuntu's staged rollout, not drift.

## What I'd do differently

- Run `ansible-inventory --host test-homelab` right after creating any `group_vars` file, before running a playbook.
- Turn on auto-save in VS Code from the start.
