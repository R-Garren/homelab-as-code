# Week 4: Ansible Vault + Docker Engine

**Date:** 18 Nov 2026 · **Status:** done

## Goal

Stop typing the sudo password (`-K`) by storing it encrypted in the repo, then install Docker Engine the way Docker documents it, entirely from a role.

## What I did

### Part A: Ansible Vault

#### The vault password file

Ansible Vault encrypts secrets with a password, and that password must never be in the repo. It lives in `~/.ansible/vault_pass.txt` on the control node (WSL):

1. `touch ~/.ansible/vault_pass.txt`, then `chmod 600`, **before** anything is written into it. The file is never readable by other users, not even for a moment.
2. The password is random: `openssl rand -base64 32 > ~/.ansible/vault_pass.txt`. Nobody ever types it, so it can be long. Redirecting into an existing file keeps its `600` mode.
3. `ansible.cfg` points at it with `vault_password_file = ~/.ansible/vault_pass.txt`. `ansible-config dump --only-changed` shows `DEFAULT_VAULT_PASSWORD_FILE` with the `~` expanded to `/home/garren/...`.

Ansible reads the password file at the start of **every** `ansible` and `ansible-playbook` command, not only when it meets an encrypted file. While the file was still empty, every run would have failed. `ansible test-homelab -m ping` returning `pong` proved the file was readable and not empty.

**Backup.** The file exists only in WSL. If WSL is rebuilt and there's no copy, every vault file becomes unreadable for good. I didn't have a password manager, so I set up Bitwarden. The vault password is stored in a Secure Note as a **hidden** field (masked on screen, still copyable), with context in the note text and master-password re-prompt turned on. Places it must not go: the OneDrive folder (including `private/`, which syncs to the cloud), the repo, or an email to myself.

#### The encrypted vault and the `vault_` prefix

`EDITOR=nano ansible-vault create inventories/test/group_vars/all/vault.yml` opened an editor on a temporary decrypted copy in `~/.ansible/tmp/`, then encrypted the result and deleted the copy on exit. I used nano rather than VS Code on purpose: VS Code's local history and backups could keep a plain-text copy of the decrypted file on disk.

The file holds one variable, `vault_ansible_become_password`, the sudo password of the `test-homelab` user. `head -1` shows `$ANSIBLE_VAULT;1.1;AES256`.

The plain `inventories/test/group_vars/all/vars.yml` points to it:

```yaml
ansible_become_password: "{{ vault_ansible_become_password }}"
```

This is the `vault_` prefix pattern: every variable name stays searchable with `grep` in a plain file, while the values stay encrypted. The quotes are required. A YAML value that starts with `{` is read as a dictionary, so `{{ … }}` without quotes is a syntax error before Jinja ever runs.

Result: `ansible-playbook playbooks/site.yml` with no `-K` got root on every task. The first run reported `changed=1` (the upgrade task found new package updates since the last run), the second `changed=0`.

#### view, edit and rekey

- `ansible-vault view` prints the decrypted file. `ansible.cfg` supplies the password, so there's no prompt.
- `ansible-vault edit` works like `create`: decrypt into the editor, re-encrypt on save.
- `ansible-vault rekey` re-encrypts with a new password. **I decided not to rotate the password this week.** It's random and has never left the control node and Bitwarden. The rotation steps are written down in ADR 0002 so they're ready when needed: a new file with mode `600`, a new random password, `rekey --new-vault-password-file`, update Bitwarden, then replace the old file. The order matters: the new password is saved in Bitwarden before the old file is overwritten, so there's always a working copy.

#### ADR 0002

[`docs/adr/0002-secrets-with-ansible-vault.md`](../adr/0002-secrets-with-ansible-vault.md) compares three options: `.env` files (v1's approach), passwordless sudo (rejected in Week 3) and Ansible Vault. It records the decision, the trust boundary it creates and the rotation steps. The guide names it `0001-…`, but ADR 0001 is already "Keep sudo-rs", so this one is 0002.

#### Stretch: a `pre-commit` hook

The Week 4 commit is the first time an encrypted file goes into the public repo. A Git hook makes sure it can never go in unencrypted.

- **Where it lives:** `scripts/hooks/pre-commit`, executable (`chmod +x`, so Git records it as `100755`). Files in `.git/hooks/` are never committed, so `git config core.hooksPath scripts/hooks` tells Git to use the folder in the repo. That setting is local to each clone.
- **What it checks:**
  - `git diff --cached --name-only --diff-filter=ACMR` lists the staged files, leaving out deleted ones, which have no content to check.
  - `grep -E '(^|/)vault\.yml$'` keeps only files named exactly `vault.yml`.
  - For each one, `git show ":$file" | head -n 1` reads the first line of the **staged** copy, which is what will actually be committed, not the copy in the working folder.
  - If that line doesn't start with `$ANSIBLE_VAULT`, the hook prints the file name and the commit is refused. Every bad file is listed before it exits with `1`.

My first draft ran `ansible-lint` on staged YAML files instead. That checks the wrong thing: a plain-text `vault.yml` is perfectly valid YAML, so lint would pass it and the password would be committed. Linting belongs in CI (Week 9).

Tests:

1. A plain-text `tmp/vault.yml`, staged with `git add -f`: commit refused with `tmp/vault.yml is not encrypted`.
2. A plain-text `test-vault.yml` was left unstaged and doesn't match the name pattern anyway.
3. The real encrypted `vault.yml`: the commit went through. `git reset --soft HEAD~1` undid the test commit and kept the file staged.

### Part B: The `docker` role

#### Following Docker's install page

Each step of [Docker's install guide for Ubuntu](https://docs.docker.com/engine/install/ubuntu/) became one task in `roles/docker/tasks/main.yml`:

| Docker's docs | Task | Module |
| --- | --- | --- |
| Uninstall old versions | Remove unofficial Docker packages | `ansible.builtin.apt`, `state: absent` |
| `apt install ca-certificates curl` | Install packages needed to add Docker's repository | `ansible.builtin.apt` |
| `install -m 0755 -d /etc/apt/keyrings` | Create the apt keyrings directory | `ansible.builtin.file` |
| `curl … -o /etc/apt/keyrings/docker.asc` and `chmod a+r` | Download Docker's GPG key | `ansible.builtin.get_url`, mode `0644` |
| `tee /etc/apt/sources.list.d/docker.sources` | Add Docker's apt repository | `ansible.builtin.deb822_repository` |
| `apt update` then `apt install docker-ce …` | Install Docker Engine | `ansible.builtin.apt`, `update_cache: true` |

Details that differ from a copy of the docs:

- **No curl.** The docs need curl because they download the key in the shell. `get_url` downloads with Python on the server, so `ca-certificates` (to trust HTTPS) is enough.
- **deb822 format.** Docker's docs now write the repository as a `docker.sources` file instead of a one-line `.list` file. `deb822_repository` writes that format directly; each parameter is one line of the docs' file, and `name: docker` becomes `docker.sources`.
- **The Ubuntu codename comes from a fact.** `suites: "{{ ansible_facts['distribution_release'] }}"` gives `resolute` on 26.04, the Ansible version of the docs' `$(. /etc/os-release && echo …)`. I used the `ansible_facts[...]` form rather than the old top-level `ansible_distribution_release`.
- **No `Architectures` line.** apt defaults to the server's own architecture (`amd64`). `ansible_facts['architecture']` would give `x86_64`, which apt doesn't accept there.
- **`update_cache: true` on the install task is required.** Without it, apt doesn't know the new repository exists and can't find `docker-ce`.
- **The package lists are variables** in `defaults/main.yml`, prefixed with the role name like `base_*`: `docker_conflicting_packages` and `docker_packages`.

Result: `Docker version 29.10.0` and `Docker Compose version v5.8.0`, both from Docker's repository.

#### One task file this time

`base` splits its tasks into five files, because updates, packages, SSH and the firewall are separate jobs and each is worth running alone with a tag. The Docker install is one chain: each step only makes sense after the one before it, and "add the repository" on its own is never useful. My rule: split when a part has its own reason to run alone, or when a file no longer fits on one screen. If the configuration tasks grow, they can move into a `configure.yml` with their own tag later.

#### Running Docker without `sudo`

- `docker_users` in `defaults/main.yml` is `[]`, and `inventories/test/group_vars/all/vars.yml` sets it to `[test-homelab]`. A fact like `ansible_user_id` can't be used here: the play runs with `become: true`, so facts are gathered as root.
- `ansible.builtin.user` loops over the list with `groups: docker` and **`append: true`**. Without `append`, Ansible would remove the user from every other group, including `sudo`.
- `ansible.builtin.meta: reset_connection` follows it. Group membership only applies to new logins, and Ansible reuses its SSH connection. A `meta` task prints no `ok` or `changed` line.

Membership in the `docker` group is effectively root: anyone in it can start a container that mounts `/` from the host. Recorded as an accepted risk in the security baseline.

#### Log rotation

Container logs grow without limit by default, and the ThinkPad's disk is small.

- `templates/daemon.json.j2` sets the `json-file` driver with `max-size` and `max-file` from `docker_log_max_size: "10m"` and `docker_log_max_file: "3"`: at most 30 MB of logs per container. Docker wants both values as strings. JSON has no comments, so this is the one managed file without an `ansible_managed` header.
- The template task uses `validate: dockerd --validate --config-file=%s`, the same idea as `sshd -t` in Week 3. A broken `daemon.json` stops Docker from starting at all.
- It notifies the `Restart Docker` handler in `handlers/main.yml`. The handler runs once at the end of the play, and only when the file changed.
- A `Create the Docker config directory` task makes sure `/etc/docker` exists. It reported `ok`: the Docker packages already create it. It stays as a safety net.

New settings only apply to containers created after the restart.

#### Ready for Week 5

- `python3-docker` is installed from Ubuntu's repository by its own task (it isn't one of Docker's packages). The `community.docker` modules used next week run in Python on the server and need it.
- `requirements.yml` now pins `community.docker` at `">=5.0.2,<6.0.0"`: 5.0.2 was a bugfix release for Docker 29, and the cap keeps out breaking changes. `ansible-galaxy collection install -r requirements.yml` had nothing to do, because the copy bundled with ansible 14.6.0 is `community.docker 5.4.0`, already in range. That's unlike `community.general` in Week 2, where the bundled 13.5.1 was too old and a separate copy went into `~/.ansible/collections`.

### Part C: The Docker + UFW trap

Prediction: UFW denies all incoming traffic except SSH, and in Week 3 `nc` to port 80 timed out, so a published container port should be blocked too.

1. `docker run -d --name trap -p 8080:80 nginx` on the VM.
2. From the desktop, `http://192.168.77.50:8080` **loaded the nginx page**, while `sudo ufw status verbose` still showed deny incoming and only `22/tcp`.
3. `sudo iptables -t nat -S | grep 8080` showed why:
   `-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80`.
   Docker rewrites the destination to the container's address before the kernel decides where the packet goes. The packet is then no longer for the VM itself, so it travels through the **FORWARD** chain instead of **INPUT**, where UFW's "deny incoming" lives. In FORWARD, Docker's chains come first (`-A FORWARD -j DOCKER-USER`, then `DOCKER-FORWARD`), and `-A DOCKER -d 172.17.0.2/32 … --dport 80 -j ACCEPT` accepts it before UFW's "deny (routed)" is reached. The ACCEPT rule matches port 80, not 8080, because the address has already been rewritten.
4. The fix: `docker run -d --name trap -p 127.0.0.1:8080:80 nginx`. From the desktop, `curl -m 5 http://192.168.77.50:8080` timed out; on the VM, `curl http://127.0.0.1:8080` still returned the nginx page. It timed out rather than being refused because, with no DNAT rule for LAN traffic, the packet goes through INPUT and UFW drops it silently, the same as port 80 in Week 3.
5. Cleaned up with `docker rm -f trap` and `docker image rm nginx`.

[`docs/security-baseline.md`](../security-baseline.md) has a new section, "Docker bypasses UFW", with the explanation and the rule for Weeks 5–7: publish on `127.0.0.1` by default; publish to the LAN only on purpose, and list those ports; containers on the host network (Pi-hole) go through UFW like any host service. The baseline also gained controls for Vault, the hook, the vault password file, Docker's signed repository and log rotation.

## Checks

- [x] `site.yml` runs with no `-K`
- [x] `head -1 inventories/test/group_vars/all/vault.yml` shows `$ANSIBLE_VAULT;1.1;AES256`
- [x] On the VM, `docker run --rm hello-world` works without `sudo`
- [x] `docker compose version` prints `Docker Compose version v5.8.0`
- [x] `docker info` shows `Logging Driver: json-file` (it shows only the driver; `max-size` and `max-file` show per container, in the `docker inspect` test below)
- [x] Second run `changed=0`
- [x] Works from a restored `01-fresh-install`: `changed=15`, then `changed=0`
- [x] `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 14 files, `production` profile
- [x] The `pre-commit` hook refuses a plain-text `vault.yml` and allows an encrypted one

## What broke and how I fixed it

1. **`head -1` seemed to hang.** Without a file name it waits for input. I pressed Ctrl+Z, which doesn't quit a program; it pauses it in the background (`Stopped`). `kill %1` ended job 1. Ctrl+C is the key that quits.
2. **`{{ vault_ansible_become_password }}` without quotes.** YAML reads a value starting with `{` as a dictionary, so it's a syntax error. My next try had two pairs of quotes (`""{{ … }}""`): YAML read `""` as an empty string followed by junk. One pair is right.
3. **`git conifg`.** A typo; Git suggested `config`. When `git config` sets a value it prints nothing, and `git config core.hooksPath` with no value prints the current setting.
4. **The first `pre-commit` hook linted YAML** instead of checking encryption. See Part A: lint passes a plain-text vault.
5. **`'docker_packages' is undefined`.** `defaults/main.yml` said `docker_packeges`. The run stopped at the install task, after the key and repository tasks had already succeeded (`changed=2`). `docker_conflicting_packages` worked, which proved the defaults file was being read and pointed at a misspelled name, the same kind of mistake as Week 3's numbers 10 and 15.
6. **`docker_users: "{{ docker_users | default([]) }}"` in `defaults/main.yml`.** Caught in review. Role defaults are the fallback when nothing else sets a variable, and this line defines the variable as itself, so without an override Ansible would stop with "recursive loop detected". The test inventory hid it because `vars.yml` wins. Defaults hold a plain value (`[]`); `| default()` is for tasks.
7. **`sudo: A terminal is required to authenticate`.** `ssh test-homelab 'sudo docker …'` runs the command without a terminal, and sudo needs one to ask for the password. `ssh -t` forces one. It also showed that sudo still needs a password.

## Evidence

### Part A

```text
$ ansible-config dump --only-changed
DEFAULT_VAULT_PASSWORD_FILE(/home/garren/code/homelab-as-code/ansible.cfg) = /home/garren/.ansible/vault_pass.txt

$ head -1 inventories/test/group_vars/all/vault.yml
$ANSIBLE_VAULT;1.1;AES256
```

`site.yml` without `-K`:

```text
test-homelab : ok=12  changed=1  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=12  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

The hook refusing a plain-text vault file:

```text
$ git commit -m "test: should be refused"
Error: tmp/vault.yml is not encrypted. Run: ansible-vault encrypt tmp/vault.yml
```

### Part B

First run after the install task was fixed, then the second run:

```text
TASK [docker : Install Docker Engine]
changed: [test-homelab]
test-homelab : ok=19  changed=1  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=19  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

The full role, second run:

```text
test-homelab : ok=23  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

```text
$ ssh test-homelab 'docker --version'
Docker version 29.10.0, build fdd4b8e

$ ssh test-homelab 'docker run --rm hello-world'
Hello from Docker!
This message shows that your installation appears to be working correctly.

$ ssh test-homelab 'docker compose version'
Docker Compose version v5.8.0

$ ssh test-homelab 'docker run --name logtest hello-world >/dev/null && docker inspect --format "{{.HostConfig.LogConfig}}" logtest && docker rm logtest'
{json-file map[max-file:3 max-size:10m]}
logtest

$ ansible-galaxy collection list community.docker
community.docker 5.4.0
```

### Part C

```text
$ sudo iptables -S | grep -i docker      (excerpt)
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-FORWARD
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
-A DOCKER ! -i docker0 -o docker0 -j DROP

$ sudo iptables -t nat -S | grep 8080
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

After re-running with `-p 127.0.0.1:8080:80`:

```text
$ curl -m 5 http://192.168.77.50:8080            (from the desktop)
curl: (28) Connection timed out after 5002 milliseconds

$ ssh test-homelab 'curl -s -m 5 http://127.0.0.1:8080 | head -4'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

### Rebuild from zero

After restoring `01-fresh-install`, the first run, then the second:

```text
PLAY [Apply base role to homelab servers]
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

PLAY [Apply docker role to homelab servers]
TASK [docker : Remove unofficial Docker packages]         ok
TASK [docker : Install packages needed to add Docker's repository] ok
TASK [docker : Create the apt keyrings directory]         ok
TASK [docker : Download Docker's GPG key]                 changed
TASK [docker : Add Docker's apt repository]               changed
TASK [docker : Install Docker Engine]                     changed
TASK [docker : Install the Docker SDK for Python …]       changed
TASK [docker : Add users to the docker group]             changed
TASK [docker : Create the Docker config directory]        ok
TASK [docker : Configure Docker log rotation]             changed
RUNNING HANDLER [docker : Restart Docker]                 changed

test-homelab : ok=25  changed=15  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
test-homelab : ok=23  changed=0   unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

The `base` part matches Week 3's rebuild exactly (8 changes). On a fresh 26.04 install, `/etc/apt/keyrings` already exists and the Docker packages create `/etc/docker`, so both directory tasks report `ok`. The whole run needed no `-K`.

- `ansible-lint playbooks/site.yml`: 0 failures, 0 warnings in 14 files, `production` profile.

## Notes for later weeks

- **Every port Portainer and NPM publish (Weeks 5 and 7) needs a decision:** `127.0.0.1`, or deliberately LAN-facing and listed in the security baseline. If a LAN-facing port must be limited by source address, the rule goes in the `DOCKER-USER` chain.
- **Restarting Docker interrupts every container.** Fine now, but from Week 6 a change to `daemon.json` would briefly take Pi-hole down. Docker's `live-restore` setting keeps containers running while the daemon restarts; worth looking at before production.
- **`site.yml` has two plays**, one per role, so facts are gathered twice. One play with `roles: [base, docker]` would gather them once.
- **The hook needs `git config core.hooksPath scripts/hooks` in every new clone.** The README says so; GitHub push protection is the second layer.
- **Vault password rotation** is documented but hasn't been run.
- **`ansible.builtin.user` creates users that don't exist**, so a typo in `docker_users` would create a new account instead of failing. Watch for `changed` on that task.
