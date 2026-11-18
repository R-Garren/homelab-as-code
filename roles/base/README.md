# Role: base

The hardened starting point for every homelab server: patches the OS and keeps it patched, installs base packages, sets the timezone and hostname, hardens SSH and turns on a deny-by-default firewall. Each control is listed in [`docs/security-baseline.md`](../../docs/security-baseline.md).

## Requirements

- Ubuntu Server 26.04 LTS on the managed host.
- ansible-core 2.21 or later on the control node.
- The `community.general` collection (`ansible-galaxy collection install -r requirements.yml`), for the `ufw` and `timezone` modules.
- Privilege escalation: apply the role with `become: true`.

## What it does

| Tag | Task file | What it does |
| --- | --- | --- |
| `updates` | `tasks/updates.yml` | Updates the apt cache, upgrades all packages, installs and configures `unattended-upgrades` (daily security updates, weekly cache cleanup, automatic reboots). |
| `packages` | `tasks/packages.yml` | Installs `base_packages`. |
| `system` | `tasks/system.yml` | Sets the timezone and sets the hostname from `inventory_hostname`. |
| `ssh` | `tasks/ssh.yml` | Writes `/etc/ssh/sshd_config.d/10-hardening.conf`, tested with `sshd -t` before it's put in place, and restarts SSH only if the file changed. |
| `firewall` | `tasks/firewall.yml` | Allows SSH with rate limiting, opens the ports in `base_ufw_allowed`, sets deny incoming and allow outgoing, then enables UFW. |

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `base_packages` | `git`, `curl`, `wget`, `vim`, `htop`, `dnsutils`, `ufw` | Packages installed on every server. |
| `base_timezone` | `Etc/UTC` | System timezone. The test inventory sets `America/New_York`. |
| `base_ssh_password_authentication` | `"no"` | sshd `PasswordAuthentication`. |
| `base_ssh_kbd_interactive_authentication` | `"no"` | sshd `KbdInteractiveAuthentication`. |
| `base_ssh_permit_root_login` | `"no"` | sshd `PermitRootLogin`. |
| `base_ssh_pubkey_authentication` | `"yes"` | sshd `PubkeyAuthentication`. |
| `base_ssh_max_auth_tries` | `3` | sshd `MaxAuthTries`. |
| `base_ufw_allowed` | `[]` | Extra ports to allow. Each entry: `{ port: "53", proto: udp, comment: DNS }`; `proto` defaults to `tcp`. SSH is always allowed separately. |
| `base_auto_upgrades_update_package_lists` | `1` | Refresh package lists every N days (`0` turns it off). |
| `base_auto_upgrades_unattended_upgrade` | `1` | Run unattended upgrades every N days (`0` turns it off). |
| `base_auto_upgrades_autoclean_interval` | `7` | Clear the downloaded package cache every N days. |
| `base_auto_upgrades_reboot` | `true` | Reboot automatically when an update needs it. |
| `base_auto_upgrades_reboot_time` | `"04:00"` | When that reboot happens. Keep it quoted: unquoted, YAML reads `04:00` as the number `240`. |

The sshd values are quoted strings on purpose: unquoted, YAML reads `no` as `false` and the template would write `False`.

## Example

```yaml
- name: Apply base role to homelab servers
  hosts: homelab
  become: true
  roles:
    - base
```

## Notes

- Removing an entry from `base_ufw_allowed` doesn't close the port: Ansible only adds UFW rules.
- Docker-published ports on bridge networks bypass UFW. See the known gaps in the security baseline.

## License

MIT
