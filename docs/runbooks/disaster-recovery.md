# Disaster recovery: targets and drill results

What happens if the server is lost, how long recovery takes, and how much data can be lost. The numbers come from a drill on the test VM, in which the server was wiped and rebuilt from code and backups. The procedure itself is in [restore.md](restore.md), Part 2.

## Summary

| | Target | Measured in the drill (2026-11-26) |
|---|---|---|
| **RTO** (time to working DNS with your data) | 15 minutes, from a freshly installed Ubuntu | **5 m 13 s** |
| **RPO** (data you can lose) | 24 hours (one backup a night, at 02:30) | 3 minutes: the restored backup was taken 3 minutes before the server was lost. No changes were made in between, so nothing was lost |
| Data restored | Pi-hole lists and settings, NPM proxy hosts and certificate, Portainer | All of it, checked below |

**What RTO includes here:** the clock starts with a bare Ubuntu Server install that already has the SSH key, and stops when Pi-hole answers DNS with the restored blocklists. Installing Ubuntu itself isn't included: in the drill, the Hyper-V checkpoint `01-fresh-install` stands in for it. Week 10 measures a real install on the ThinkPad.

## The drill

### Scenario

The test VM (`test-homelab`, 192.168.77.50) was running Pi-hole, Unbound, Nginx Proxy Manager and Portainer, with two changes made that morning that exist only in the containers' data, not in Git:

- **Pi-hole:** a second blocklist (HaGeZi Light, `DR drill`), bringing the total to 163,522 domains, and an exact deny entry for `drill-test.example.com`.
- **NPM:** a fourth proxy host, `drill.home.arpa` → `http://172.20.0.1:8081`, HTTP only.

A manual backup ran at 11:09 (stamp `2026-11-26_1109`) and `backup-pull.yml` copied it to WSL. Then the server was "lost": the VM was rolled back to the checkpoint `01-fresh-install` (bare Ubuntu, no Docker, no data). The state before the drill was kept as checkpoint `03-before-dr-drill`, as a safety net.

### Timeline

Each step was logged with a timestamp from WSL. Times are from the start of the drill.

| Clock | Elapsed | Step |
|---|---|---|
| 11:12:10 | 0:00 | Server lost (checkpoint `01-fresh-install` applied). Checked bare: `docker: command not found`, DNS connection refused |
| 11:12:40 | 0:30 | `ansible-playbook playbooks/site.yml` started |
| 11:15:00 | 2:50 | `site.yml` finished: `ok=60 changed=35 failed=0` (2 m 20 s) |
| 11:15:51 | 3:41 | All four containers up (Portainer deploys the stacks in the background) |
| 11:16:06 | 3:56 | DNS answering, but with empty lists: `drill-test.example.com` not blocked |
| 11:17:23 | **5:13** | **Pi-hole volume restored, DNS answering with the restored lists** (`drill-test.example.com` → `0.0.0.0`). **RTO stop** |
| 11:17:41 | 5:31 | NPM and Portainer volumes restored |
| 11:19:53 | 7:43 | Every check below passed |

`site.yml` took 2 m 20 s, close to the 2 m 05 s from-zero run in [ADR 0007](../adr/0007-test-vm-on-hyper-v.md); the difference is the new `backup` role and a day's worth of package updates.

### Checks after the restore

| Check | Result |
|---|---|
| `dig @192.168.77.50 example.com` | Answers |
| `dig @192.168.77.50 drill-test.example.com` | `0.0.0.0`: the deny entry is back |
| Pi-hole dashboard | 2 lists, 163,522 domains on lists, 1 deny entry: the same as before the drill |
| NPM proxy hosts | All four online, including `drill.home.arpa`. The three original hosts still show their creation date of 2026-11-25, so they came from the backup, not from being created again |
| `curl -sI -H 'Host: drill.home.arpa' http://192.168.77.50/admin/` | `302 Found` from NPM (`openresty`) to Pi-hole's login page |
| `https://npm.home.arpa`, `https://pihole.home.arpa`, `https://portainer.home.arpa` | Load with the private CA's certificate |
| Portainer | Log in as `admin` works; stacks `dns` and `proxy` are Git-managed (`refs/heads/main`, commit `f5b90f2`) with full control |
| File owners in `dns_pihole_data` | Unchanged (`1000:1000`); the only differences were files the backup brought back |
| `systemctl list-timers homelab-backup.timer` | The backup timer is back, next run 02:30 |
| `systemctl --failed` | 0 units |
| Week 6 and Week 7 checklists | Passed |

### What the drill showed

- **The backups are complete.** Everything that isn't in Git came back: Pi-hole's lists and deny entries, NPM's proxy hosts and uploaded certificate, Portainer's database. NPM's proxy hosts still aren't in Git ([ADR 0005](../adr/0005-private-ca.md)), but they are now recoverable.
- **Code and backups split the work as designed.** `site.yml` rebuilt everything Ansible owns, and the backups only had to carry state. DNS was already answering at 3:56, before any restore; the restore added the data 1 m 17 s later.
- **Restoring `portainer_data` was safe** on top of the stacks `site.yml` had just created: the IDs and names matched, and Portainer kept full control of both stacks.
- **Two runbook mistakes were found and fixed** in [restore.md](restore.md): `ssh host docker ps --format '…'` loses its quotes on the way to the server (the whole remote command must be quoted), and the explanation of the owner `diff` only covered files newer than the backup, not files the backup brings back.

## What recovery depends on

The drill proved recovery works while the desktop (the control node) is fine. These are the things that must survive for it to work, and where each one lives today:

| Needed | Where it is | Backed up? |
|---|---|---|
| The code | GitHub, `R-Garren/homelab-as-code` | Yes (GitHub, plus the WSL clone) |
| Vault password (`~/.ansible/vault_pass.txt`) | WSL | Yes, in Bitwarden |
| SSH key for the server (`homelab_v2`) | WSL | If it's lost, make a new key and add it during the Ubuntu install |
| The archives | `/var/backups/homelab/` on the server, `~/homelab-backups/<host>/` on WSL | Two copies, but both on the desktop's SSD while testing on the VM |
| The private CA (`~/.homelab-pki/`) | WSL | **No.** NPM's volume carries the server certificate and its key, so a restore doesn't need the CA. A new certificate would: losing the CA means building a new one and importing its `ca.crt` on every device again |

## Open items

- **Off-site copy (the last "1" of 3-2-1).** In production, the server and WSL are separate machines, which gives two copies on two devices. A third, off-site copy (OneDrive, for example) needs the archives encrypted first, because they hold password hashes and NPM's private key.
- **Back up `~/.homelab-pki/`** the same way, encrypted, so losing the desktop doesn't mean a new CA.
- **Measure a real RTO in Week 10:** Ubuntu install on the ThinkPad + `site.yml` + restore.

## Running the drill again

1. Make a change that exists only in data (a Pi-hole list, an NPM proxy host), then run a backup (`sudo systemctl start homelab-backup.service` on the server) and `ansible-playbook playbooks/backup-pull.yml`.
2. In Hyper-V Manager, apply the checkpoint `01-fresh-install` with **Create Checkpoint and Apply**.
3. Log each step with a timestamp from WSL:
   ```bash
   mark() { echo "$(date +%T) $*" | tee -a ~/dr-drill-$(date +%F).log; }
   mark "START: server lost"
   ```
4. Follow [restore.md](restore.md), Part 2, with a `mark` after `site.yml`, after the Pi-hole restore (RTO stop) and after the last check.
5. Add the results to this file.
