# Security baseline

Every homelab server gets the same hardened starting point from three Ansible roles: `base` (`roles/base/`) hardens the operating system, `docker` (`roles/docker/`) installs Docker Engine, and `portainer` (`roles/portainer/`) runs Portainer CE. This page lists each control, where it lives in the code, how to check it on a running server, and the Security+ concept it maps to.

**Applies to:** the test VM `test-homelab` today, and the production ThinkPad from Week 10. **Last verified:** 18 Nov 2026 ([Week 5 build log](build-log/week-05.md)).

## Controls

| Control | How it's implemented | How to verify it | Security+ concept |
| --- | --- | --- | --- |
| SSH accepts keys only | `PasswordAuthentication no`, `KbdInteractiveAuthentication no` and `PubkeyAuthentication yes` in `/etc/ssh/sshd_config.d/10-hardening.conf` (template `sshd_hardening.conf.j2`). The `10-` prefix makes it win over the installer's `50-cloud-init.conf`, because sshd keeps the first value it reads. | `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <user>@<host>` is refused with `Permission denied (publickey)`. On the server, `sudo sshd -T` shows `passwordauthentication no`. | Hardening; authentication (something you have) |
| No root login over SSH | `PermitRootLogin no` in the same file. Admins log in as their own user and use sudo, so every action is tied to a named account. | `ssh root@<host>` is refused. `sudo sshd -T` shows `permitrootlogin no`. | Least privilege; accountability |
| Fewer password guesses per connection | `MaxAuthTries 3` (the default is 6). | `sudo sshd -T` shows `maxauthtries 3`. | Hardening |
| A broken SSH config can't be applied | The template task uses `validate: /usr/sbin/sshd -t -f %s`: the new file is tested by sshd itself before it replaces the old one. | On the test VM only: put a misspelled setting in the template and do a real run. The task fails with sshd's `Bad configuration option` error and the server's file is unchanged. (`validate` doesn't run in `--check` mode.) | Change management |
| Host firewall: deny inbound by default | UFW with default deny incoming and allow outgoing (`tasks/firewall.yml`). The firewall is switched on last, after the SSH rule exists. **Doesn't cover ports Docker publishes**: see [Docker bypasses UFW](#docker-bypasses-ufw). | `sudo ufw status verbose` shows `Status: active` and `Default: deny (incoming), allow (outgoing)`. From another machine, `nc -zv -w 3 <host> 80` times out instead of being refused. | Implicit deny; defense in depth |
| SSH rate limiting | UFW `limit` rule on `22/tcp`: an address that opens 6 or more connections in 30 seconds is blocked. | `sudo ufw status verbose` shows `22/tcp LIMIT IN` for IPv4 and IPv6. | Brute-force mitigation; defense in depth |
| Only the ports a service needs are open | Extra ports come only from the `base_ufw_allowed` list, which is empty today. SSH has its own task, so changing the list can't lock out SSH. | `sudo ufw status` lists only `22/tcp` (until Week 6 adds DNS, DHCP and the Pi-hole web UI). | Attack surface reduction |
| Automatic security updates | `unattended-upgrades` installed and enabled: package lists refresh and security updates install daily (`20auto-upgrades`). Downloaded packages are cleaned weekly. | `apt-config dump \| grep -E 'Periodic\|Automatic-Reboot'` shows `Update-Package-Lists "1"` and `Unattended-Upgrade "1"`. `sudo unattended-upgrade --dry-run --debug` lists the allowed origins (`resolute-security` and the ESM security sources). | Patch management |
| Updates that need a reboot take effect | `Automatic-Reboot "true"` at `04:00` (`52unattended-upgrades-local`). See the decision below. | `apt-config dump` shows `Automatic-Reboot "true"` and `Automatic-Reboot-Time "04:00"`. | Patch management; availability |
| Becoming root needs a password | No `NOPASSWD` rule for the Ansible user. Playbooks get the sudo password from Ansible Vault (`vault_ansible_become_password`), so `-K` is no longer needed. | `sudo -k; sudo true` on the server asks for the password. | Least privilege; defense in depth |
| Secrets are encrypted in the repo | Every secret lives in `inventories/<env>/group_vars/all/vault.yml`, encrypted with Ansible Vault (AES-256). Plain `vars.yml` only points to it with `vault_`-prefixed names. See [ADR 0002](adr/0002-secrets-with-ansible-vault.md). | `head -1` on each `vault.yml` shows `$ANSIBLE_VAULT;1.1;AES256`. | Encryption at rest; secrets management; confidentiality |
| An unencrypted vault file can't be committed | Git `pre-commit` hook (`scripts/hooks/pre-commit`) checks the staged copy of every `vault.yml` and refuses the commit if its first line isn't `$ANSIBLE_VAULT`. Turned on per clone with `git config core.hooksPath scripts/hooks`. | Stage a plain-text `vault.yml` and commit: refused with `<file> is not encrypted`. An encrypted one commits normally. | Preventive control; data loss prevention |
| The vault password stays off the repo | Random 32-byte password (`openssl rand -base64 32`) in `~/.ansible/vault_pass.txt` on the control node, mode `600`. `ansible.cfg` points to it. A backup copy is in a password manager. `.gitignore` blocks `vault_pass*`. | `ls -l ~/.ansible/vault_pass.txt` shows `-rw-------`. `ansible-config dump --only-changed` shows `DEFAULT_VAULT_PASSWORD_FILE` outside the repo. | Key management; separation of key and data |
| Docker comes from Docker's signed repository | Ubuntu's unofficial Docker packages are removed. Docker's GPG key is stored in `/etc/apt/keyrings/docker.asc`, and the repository (`docker.sources`) only trusts packages signed by that key (`Signed-By`). | `apt-cache policy docker-ce` shows `download.docker.com` as the source. `cat /etc/apt/sources.list.d/docker.sources` shows `Signed-By: /etc/apt/keyrings/docker.asc`. | Supply chain security; integrity |
| Portainer runs a pinned, supported release | `portainer_version: "2.45.3"`, the current LTS release, in `roles/portainer/defaults/main.yml`. Never `latest`, so a rebuild runs the version that was tested. Upgrades are a deliberate change to that one variable. | `docker inspect --format '{{.Config.Image}}' portainer` shows `portainer/portainer-ce:2.45.3`. The Portainer footer shows `Community Edition 2.45.3 LTS`. | Patch management; change management; supply chain security |
| Portainer publishes only the port it needs | Only the HTTPS UI is published (`9443`). Port `8000`, used by Edge agents, isn't published, and Edge Compute was left off during setup. Portainer's data is in the named volume `portainer_data`, not a host folder. | `sudo ss -tulpn` shows `docker-proxy` on 9443 and nothing on 8000. `docker ps` shows only `0.0.0.0:9443->9443/tcp` for `portainer`. | Attack surface reduction |
| Portainer's admin account can't be claimed remotely | Built into Portainer 2.45: the first-run page needs a one-time setup token that Portainer prints only to its own log, and it locks the setup page after a few minutes. The admin password must be at least 12 characters and is kept in Bitwarden. | On a fresh install, `https://<host>:9443` asks for the setup token. `docker logs portainer 2>&1 \| grep -i token` on the server shows it. | Secure defaults; authentication |
| Container logs can't fill the disk | `/etc/docker/daemon.json` sets the `json-file` driver with `max-size 10m` and `max-file 3`, so each container keeps at most 30 MB of logs. The file is checked with `dockerd --validate` before it's put in place. | `docker inspect --format '{{.HostConfig.LogConfig}}' <container>` shows `{json-file map[max-file:3 max-size:10m]}`. | Availability; resource exhaustion |
| The server matches the code | Every setting above is applied by `site.yml`. Managed files carry an `Ansible managed` header (except `daemon.json`, because JSON has no comments). A second run reports `changed=0`, so any change means something drifted. | `ansible-playbook playbooks/site.yml --check --diff` reports `changed=0` and shows no diffs. | Configuration management; secure baseline |
| Changes are tested before production | Every change runs on the test VM first, with VirtualBox snapshots as rollback points, and is rebuilt from the `01-fresh-install` snapshot before release. | Restore `01-fresh-install`, run `site.yml`, and the result matches a long-running server. | Change management |

## Docker bypasses UFW

**Finding (18 Nov 2026):** a container started with `docker run -d --name trap -p 8080:80 nginx` was reachable from another machine at `http://192.168.77.50:8080`, while `ufw status` still showed deny incoming with only `22/tcp` allowed.

**Why:** a published port is handled by Docker's own iptables rules, not by UFW.

1. Docker adds a `nat` rule for each published port: `-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80`. It rewrites the destination to the container's internal address before the kernel decides where the packet goes.
2. The packet is no longer addressed to the server itself, so it goes through the **FORWARD** chain, not **INPUT**. UFW's "deny incoming" rules live in INPUT and never see it.
3. In FORWARD, Docker's chains come first (`-A FORWARD -j DOCKER-USER`, then `DOCKER-FORWARD`), and `-A DOCKER -d 172.17.0.2/32 … --dport 80 -j ACCEPT` accepts the packet before UFW's "deny (routed)" rules are reached.

**Fix tested:** with `-p 127.0.0.1:8080:80` the port is published on the loopback address only. From the LAN, `curl -m 5 http://192.168.77.50:8080` timed out; on the server, `curl http://127.0.0.1:8080` still returned the nginx page.

**Rules for every container from Week 5 on:**

| Kind of port | Rule | Firewall that applies |
| --- | --- | --- |
| Only used on the server, or only reached through the reverse proxy | Publish on `127.0.0.1` (`127.0.0.1:<host>:<container>`), or don't publish it at all and use a Docker network. | None needed: not reachable from the LAN |
| Must be reachable from the LAN (Portainer in Week 5, Nginx Proxy Manager 80/443 in Week 7) | Publish deliberately and list it in the table below. UFW does **not** protect it. If it must be limited by source address, the rule goes in the `DOCKER-USER` chain, which Docker never changes and checks first. | Docker's iptables rules (UFW is bypassed) |
| Container on the host network (Pi-hole) | Behaves like a normal host service. | UFW, through `base_ufw_allowed` |

**Ports published to the LAN:**

| Port | Container | Published as | Why it's LAN-facing | Source filter | Until |
| --- | --- | --- | --- | --- | --- |
| `9443/tcp` | `portainer` (HTTPS UI) | `0.0.0.0:9443->9443/tcp` | The admin UI has to be reached from the desktop, and there's no reverse proxy yet. | None. No `DOCKER-USER` rule; decided 18 Nov 2026. | Week 7, when Nginx Proxy Manager fronts Portainer over a shared Docker network and 9443 is no longer published. |

**How to check:** `docker ps` shows each container's published address in the PORTS column (`127.0.0.1:8080->80/tcp` is local only; `0.0.0.0:8080->80/tcp` is open to the LAN). `sudo iptables -t nat -S DOCKER` lists every DNAT rule.

**Not chosen:** turning off Docker's iptables management (`"iptables": false` in `daemon.json`) breaks container networking and needs hand-written NAT rules. Third-party scripts that rewrite UFW's rules for Docker add something else to maintain. Binding to `127.0.0.1` plus a short, documented list of LAN-facing ports needs neither.

## Decisions and accepted risks

| Decision | Why | Risk accepted, and what limits it |
| --- | --- | --- |
| Keep `50-cloud-init.conf` (it sets `PasswordAuthentication yes`) | Our `10-hardening.conf` is read first and wins, so the hardening doesn't depend on cloud-init never rewriting its file. | None while the `10-` prefix stays lower than `50-`. `sshd -T` shows the effective value. |
| No `AllowUsers` yet | One wrong username there locks everyone out, and key-only login with no root already closes the main paths. | Any account with an authorized key can log in. Revisit once the user list is settled. |
| No passwordless sudo; the become password lives in Ansible Vault | A stolen SSH key alone shouldn't give root. Vault removes typing `-K` without removing the password. | The SSH key and the vault password file are both on the control node, so someone who takes over my WSL account gets root on the server. The control node is part of the trust boundary. |
| Vault password rotation documented, not yet run | The password is random and has never left the control node and the password manager. | Rotate it if it may have leaked, a device holding it is lost, or someone who knew it no longer should. Steps in [ADR 0002](adr/0002-secrets-with-ansible-vault.md). |
| The admin user is in the `docker` group | Day-to-day `docker` commands work without `sudo`. | Members of the `docker` group are effectively root: they can start a container that mounts `/` from the host. Only the admin user is in `docker_users`. |
| Automatic reboots on, at 04:00 | A kernel or library fix doesn't protect the server until it reboots. Upgrades run between about 06:00 and 07:00, so the reboot lands at 04:00 the next morning, when nobody is using the network. | From Week 10 this server is the home's only DNS and DHCP server: a reboot is a short outage, and a failed boot is a full one. Fallback: the Pi-hole-down runbook (turn DHCP back on at the BGW320). A fix that needs a reboot can wait up to about a day. |
| Portainer mounts the Docker socket (`/var/run/docker.sock`) | Portainer manages Docker through the socket; without it there's nothing to manage. | Anyone who is a Portainer admin, or who exploits a Portainer vulnerability, is effectively root on the host: they can start a privileged container that mounts `/`. Limits: a strong admin password, a pinned and patched LTS release, and the UI reachable only from the LAN. Mounting the socket read-only (`:ro`) wouldn't help, because it doesn't stop API calls through the socket. |
| Port 9443 published to the LAN without a source filter | It's needed until Week 7, and the home network is one flat subnet, so a filter would only exclude the few devices that aren't the desktop. A `DOCKER-USER` rule would also need a DHCP reservation for the desktop. | Any device on the LAN can reach Portainer's login page. Temporary: listed in [Ports published to the LAN](#docker-bypasses-ufw) and removed in Week 7. |
| Self-signed certificate on Portainer until Week 7 | Portainer generates it on first start. A private CA comes in Week 7. | The browser can't tell Portainer's certificate from an attacker's on the LAN, and clicking through the warning becomes a habit. Limited to the LAN; Week 7 replaces it with a certificate signed by the private CA. |
| Stay on Portainer CE, frozen on the 2.45 LTS line | Portainer announced in September 2026 that there will be no CE build of 3.x; 2.45 LTS still gets security and bug fixes. See [ADR 0003](adr/0003-ansible-owns-host-portainer-owns-apps.md). | If CE fixes stop, Portainer becomes unpatched software with root-level access. The stacks are plain Compose files in Git, so they can move to another manager without rewriting the apps. |
| Only the security pocket updates automatically | That's Ubuntu's default: security fixes are small and targeted. | Non-security updates wait for the next `site.yml` run, whose upgrade task applies everything. |

## Known gaps

- **LAN-facing container ports aren't filtered by source.** Port 9443 (Portainer) is open to every device on the LAN until Week 7, because UFW doesn't apply. `DOCKER-USER` rules can close this if needed; decided per port.
- **The Portainer admin password isn't managed by Ansible yet.** It was set by hand on first login and lives only in Bitwarden. Week 8 creates the admin through Portainer's API with the password from Ansible Vault.
- **The `pre-commit` hook only protects clones where it's turned on.** `core.hooksPath` is a local setting, so a fresh clone has no hook until `git config core.hooksPath scripts/hooks` is run. GitHub push protection is the second layer.
- **Removing a port from `base_ufw_allowed` doesn't close it.** Ansible only adds UFW rules; closing one needs a task with `delete: true`.
- **SSH is reachable from the whole LAN.** No source restriction yet, and the home network is one flat subnet until the VLAN segmentation lab in Week 11.
- **No intrusion detection or central logging yet.** UFW logs blocked traffic locally (`Logging: on (low)`). Suricata comes in Week 11; Wazuh is paused until after v2.x.

## Planned controls

| Week | Control |
| --- | --- |
| 7 | HTTPS for local services, signed by a private CA; Portainer reached only through Nginx Proxy Manager, with 9443 no longer published |
| 8 | Portainer admin account and Git stacks created by Ansible through Portainer's API, with secrets from Vault |
| 9 | Backups, CI checks (`ansible-lint` on every push) and a disaster-recovery drill |
| 11 | VLAN segmentation with default deny between zones; Suricata IDS (OPNsense lab) |
| 12 | WireGuard for remote access, the only port open to the internet |
