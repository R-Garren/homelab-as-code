# ADR 0008: Back up state, not config, with a nightly volume archive

**Status:** Accepted · **Date:** 26 Nov 2026

## Context

From Week 10 the ThinkPad is the home's only DNS and DHCP server. Everything Ansible owns (packages, SSH, firewall, Docker, Portainer itself, Unbound's config) and everything Portainer deploys from Git (`stacks/`) can be rebuilt by `site.yml` in about 2 minutes ([ADR 0007](0007-test-vm-on-hyper-v.md)). What `site.yml` can't rebuild is the data the apps keep in their Docker volumes:

| Volume | Holds | Recreated by code? |
| --- | --- | --- |
| `dns_pihole_data` | Blocklists, allow/deny entries, settings, query history, DHCP leases and reservations | No |
| `proxy_npm_data` | NPM's proxy hosts, the uploaded certificate and its key, NPM's admin account | No ([ADR 0005](0005-private-ca.md)) |
| `proxy_npm_letsencrypt` | Empty (private CA, no Let's Encrypt) | Nothing to recreate |
| `portainer_data` | Portainer's users, settings, Git source and stacks | Mostly, by `portainer_stacks` since Week 8 ([ADR 0006](0006-portainer-admin-setup-token.md)). Hand-made changes aren't |

Portainer prefixes a stack's volumes with the stack name, so these are the names `docker volume ls` shows, not the ones in the compose files.

The archives hold secrets: Pi-hole's password hash, NPM's admin account and the server certificate's private key. Wherever they go has to be as protected as the server.

## Options considered

| Option | How it works | Downside |
| --- | --- | --- |
| A. Back up the whole disk (image) | `dd`, Clonezilla or a VM checkpoint | Huge, mostly things Git and Ansible already rebuild. Needs downtime or a snapshot-capable filesystem. A checkpoint only exists for the test VM, not the ThinkPad |
| B. Each app's own export | Pi-hole Teleporter, Portainer's backup button, nothing for NPM | Three different methods, two of them clicked by hand. NPM has no export at all |
| C. Back up everything under `/var/lib/docker` | One `tar` of Docker's data directory | Includes images and container layers that are pulled again anyway. Restoring it means stopping Docker entirely, and it couples the backup to Docker's internal layout |
| **D. Archive only the named volumes, nightly** | A script stops the containers that write to them, tars each volume through a throwaway container, starts them again. A systemd timer runs it | A short outage every night. Unbound's in-memory cache isn't kept (it refills by itself) |
| E. A backup tool (restic, Borg, Duplicati) | Deduplicated, encrypted, often with off-site targets | Another piece of software and key to manage. Worth it for an off-site copy later, more than this lab needs for a local one |

## Decision

**Option D.** The `backup` role ([README](../../roles/backup/README.md)) installs:

| Part | Choice | Why |
| --- | --- | --- |
| What | The four named volumes above (`backup_volumes`) | **Back up state, not config.** Everything else is in Git or rebuilt by Ansible, so a restore is `site.yml` first, then the volumes |
| How | `homelab-backup.sh`: `docker stop pihole npm portainer`, then for each volume a throwaway `alpine:3.24.4` container mounts it read-only and runs `tar -czf`, then `docker start` | Stopping first means the SQLite databases (`gravity.db`, NPM's and Portainer's) are copied in a consistent state. A `trap … EXIT` starts the containers again even if `tar` fails, so a broken backup never leaves DNS down. The helper image is pulled when the role runs, so 02:30 doesn't depend on Docker Hub |
| When | systemd timer, `OnCalendar=*-*-* 02:30:00`, `Persistent=true` | A timer logs to the journal (`journalctl -u homelab-backup.service`), shows its next and last run in `systemctl list-timers`, and catches up a missed run at the next boot. 02:30 is clear of the 04:00 automatic reboot |
| Where | `/var/backups/homelab/`, folder `0700`, archives `0600`, owned by root | The archives are secrets |
| How long | 7 days (`find -mtime +7 -delete`), which keeps 8–9 runs because `find` rounds ages down | A week covers "it broke a few days ago and I just noticed" without filling the disk |
| Off the server | `playbooks/backup-pull.yml` fetches the newest run to `~/homelab-backups/<host>/` on WSL (`0700`), outside the repo and outside OneDrive | A backup on the same disk as the data doesn't survive losing that disk |
| Restore | [`docs/runbooks/restore.md`](../runbooks/restore.md): one volume in place, or every volume after a rebuild | Tested both ways on the test VM |

**Targets** ([disaster-recovery.md](../runbooks/disaster-recovery.md)): RPO 24 hours (one run a night), RTO 10 minutes from a freshly installed Ubuntu to DNS answering with the restored lists. The drill on 26 Nov 2026 measured 5 m 13 s.

## Consequences

- **About 6 seconds without DNS every night at 02:30,** while Pi-hole is stopped. NPM and Portainer are down for the same few seconds. Unbound keeps running.
- **Restores are proven, not assumed.** An in-place restore undid a disabled NPM proxy host, and the disaster-recovery drill rebuilt the VM from `01-fresh-install` and restored every volume, with file owners unchanged.
- **NPM's clicked configuration is now recoverable** even though it still isn't in Git. ADR 0005's gap is smaller, not closed.
- **Pi-hole's DHCP reservations stay out of the public repo.** They hold MAC addresses, so they live only in `dns_pihole_data` and its archives. The DHCP scope (range, gateway, lease time) is in `inventories/prod` as variables.
- **3-2-1 is two copies on one machine while testing.** The VM's disk and WSL are both on the desktop's SSD, so the drill proves the process, not the rule. In production the server and WSL are separate machines: two copies on two devices.
- **No off-site copy yet.** OneDrive would be the "1" of 3-2-1, but only after the archives are encrypted. Until then nothing leaves the house.
- **The private CA (`~/.homelab-pki/`) isn't covered.** It lives on WSL, not on the server, and its key is in Bitwarden. A restore doesn't need it (the server certificate is in `proxy_npm_data`); issuing a new certificate does.
- **Adding a stateful service means adding its volume** to `backup_volumes`, and its container to `backup_stop_containers` if it writes a database. WireGuard in Week 11 is the next candidate.
