# Security baseline

Every homelab server gets the same hardened starting point from the Ansible `base` role (`roles/base/`). This page lists each control, where it lives in the code, how to check it on a running server, and the Security+ concept it maps to.

**Applies to:** the test VM `test-homelab` today, and the production ThinkPad from Week 10. **Last verified:** 18 Nov 2026 ([Week 3 build log](build-log/week-03.md)).

## Controls

| Control | How it's implemented | How to verify it | Security+ concept |
| --- | --- | --- | --- |
| SSH accepts keys only | `PasswordAuthentication no`, `KbdInteractiveAuthentication no` and `PubkeyAuthentication yes` in `/etc/ssh/sshd_config.d/10-hardening.conf` (template `sshd_hardening.conf.j2`). The `10-` prefix makes it win over the installer's `50-cloud-init.conf`, because sshd keeps the first value it reads. | `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <user>@<host>` is refused with `Permission denied (publickey)`. On the server, `sudo sshd -T` shows `passwordauthentication no`. | Hardening; authentication (something you have) |
| No root login over SSH | `PermitRootLogin no` in the same file. Admins log in as their own user and use sudo, so every action is tied to a named account. | `ssh root@<host>` is refused. `sudo sshd -T` shows `permitrootlogin no`. | Least privilege; accountability |
| Fewer password guesses per connection | `MaxAuthTries 3` (the default is 6). | `sudo sshd -T` shows `maxauthtries 3`. | Hardening |
| A broken SSH config can't be applied | The template task uses `validate: /usr/sbin/sshd -t -f %s`: the new file is tested by sshd itself before it replaces the old one. | On the test VM only: put a misspelled setting in the template and do a real run. The task fails with sshd's `Bad configuration option` error and the server's file is unchanged. (`validate` doesn't run in `--check` mode.) | Change management |
| Host firewall: deny inbound by default | UFW with default deny incoming and allow outgoing (`tasks/firewall.yml`). The firewall is switched on last, after the SSH rule exists. | `sudo ufw status verbose` shows `Status: active` and `Default: deny (incoming), allow (outgoing)`. From another machine, `nc -zv -w 3 <host> 80` times out instead of being refused. | Implicit deny; defense in depth |
| SSH rate limiting | UFW `limit` rule on `22/tcp`: an address that opens 6 or more connections in 30 seconds is blocked. | `sudo ufw status verbose` shows `22/tcp LIMIT IN` for IPv4 and IPv6. | Brute-force mitigation; defense in depth |
| Only the ports a service needs are open | Extra ports come only from the `base_ufw_allowed` list, which is empty today. SSH has its own task, so changing the list can't lock out SSH. | `sudo ufw status` lists only `22/tcp` (until Week 6 adds DNS, DHCP and the Pi-hole web UI). | Attack surface reduction |
| Automatic security updates | `unattended-upgrades` installed and enabled: package lists refresh and security updates install daily (`20auto-upgrades`). Downloaded packages are cleaned weekly. | `apt-config dump \| grep -E 'Periodic\|Automatic-Reboot'` shows `Update-Package-Lists "1"` and `Unattended-Upgrade "1"`. `sudo unattended-upgrade --dry-run --debug` lists the allowed origins (`resolute-security` and the ESM security sources). | Patch management |
| Updates that need a reboot take effect | `Automatic-Reboot "true"` at `04:00` (`52unattended-upgrades-local`). See the decision below. | `apt-config dump` shows `Automatic-Reboot "true"` and `Automatic-Reboot-Time "04:00"`. | Patch management; availability |
| Becoming root needs a password | No `NOPASSWD` rule for the Ansible user. Playbooks run with `-K` and ask for the sudo password. | `sudo -k; sudo true` on the server asks for the password. | Least privilege; defense in depth |
| The server matches the code | Every setting above is applied by `site.yml`. Managed files carry an `Ansible managed` header. A second run reports `changed=0`, so any change means something drifted. | `ansible-playbook playbooks/site.yml -K --check --diff` reports `changed=0` and shows no diffs. | Configuration management; secure baseline |
| Changes are tested before production | Every change runs on the test VM first, with VirtualBox snapshots as rollback points, and is rebuilt from the `01-fresh-install` snapshot before release. | Restore `01-fresh-install`, run `site.yml`, and the result matches a long-running server. | Change management |

## Decisions and accepted risks

| Decision | Why | Risk accepted, and what limits it |
| --- | --- | --- |
| Keep `50-cloud-init.conf` (it sets `PasswordAuthentication yes`) | Our `10-hardening.conf` is read first and wins, so the hardening doesn't depend on cloud-init never rewriting its file. | None while the `10-` prefix stays lower than `50-`. `sshd -T` shows the effective value. |
| No `AllowUsers` yet | One wrong username there locks everyone out, and key-only login with no root already closes the main paths. | Any account with an authorized key can log in. Revisit once the user list is settled. |
| No passwordless sudo | A stolen SSH key alone shouldn't give root. | Typing `-K` on every run. Week 4's Ansible Vault can hold the become password instead. |
| Automatic reboots on, at 04:00 | A kernel or library fix doesn't protect the server until it reboots. Upgrades run between about 06:00 and 07:00, so the reboot lands at 04:00 the next morning, when nobody is using the network. | From Week 10 this server is the home's only DNS and DHCP server: a reboot is a short outage, and a failed boot is a full one. Fallback: the Pi-hole-down runbook (turn DHCP back on at the BGW320). A fix that needs a reboot can wait up to about a day. |
| Only the security pocket updates automatically | That's Ubuntu's default: security fixes are small and targeted. | Non-security updates wait for the next `site.yml` run, whose upgrade task applies everything. |

## Known gaps

- **Docker skips UFW (from Weeks 4–5).** Docker writes its own iptables rules, and ports it publishes for containers on bridge networks (Portainer, Nginx Proxy Manager) bypass UFW completely. Pi-hole uses the host network, so its ports do go through UFW. This needs its own control before Week 10.
- **Removing a port from `base_ufw_allowed` doesn't close it.** Ansible only adds UFW rules; closing one needs a task with `delete: true`.
- **SSH is reachable from the whole LAN.** No source restriction yet, and the home network is one flat subnet until the VLAN segmentation lab in Week 11.
- **No intrusion detection or central logging yet.** UFW logs blocked traffic locally (`Logging: on (low)`). Suricata comes in Week 11; Wazuh is paused until after v2.x.

## Planned controls

| Week | Control |
| --- | --- |
| 4 | Secrets in Ansible Vault; vault password file outside the repo |
| 7 | HTTPS for local services, signed by a private CA |
| 9 | Backups, CI checks (`ansible-lint` on every push) and a disaster-recovery drill |
| 11 | VLAN segmentation with default deny between zones; Suricata IDS (OPNsense lab) |
| 12 | WireGuard for remote access, the only port open to the internet |
