# Role: backup

Backs up the server's state every night: a systemd timer runs a script that archives each Docker volume into `/var/backups/homelab/`, then deletes archives older than a week. Only state that can't be regenerated is backed up. The OS, Docker, configs and compose files come back from Ansible and Git, so they aren't in the archives. `playbooks/backup-pull.yml` copies the newest archives off the server, and [`docs/runbooks/restore.md`](../../docs/runbooks/restore.md) puts them back.

## Requirements

- Ubuntu Server 26.04 LTS on the managed host.
- ansible-core 2.21 or later on the control node, with the `community.docker` collection (see `requirements.yml`).
- Privilege escalation: apply the role with `become: true`.
- Docker installed by the `docker` role. Apply this role last in `site.yml`: it doesn't need the stacks to exist, but the script backs up their volumes.

## What it does

All tasks are in `tasks/main.yml`, in this order:

| Task | What it does |
| --- | --- |
| Create the backup directory | `backup_dir`, owned by root, mode `0700`. The archives hold password hashes and NPM's private key. |
| Pull the backup helper image | Pulls `backup_helper_image` now, so the nightly run doesn't depend on Docker Hub. |
| Install the backup script | `/usr/local/bin/homelab-backup.sh` from `templates/homelab-backup.sh.j2`, mode `0750`, checked with `bash -n` before it's put in place. |
| Install the backup service unit | `/etc/systemd/system/homelab-backup.service`: a `oneshot` service that runs the script. Notifies `Reload systemd`. |
| Install the backup timer unit | `/etc/systemd/system/homelab-backup.timer`: runs the service at `backup_schedule`. Notifies `Reload systemd` and `Restart backup timer`. |
| Apply pending systemd reloads now | `meta: flush_handlers`, so systemd knows the new units before the next task enables the timer. |
| Enable and start the backup timer | Enables and starts the **timer**. The service is never enabled: the timer starts it. |

Handlers, in the order they run: `Reload systemd` (`daemon-reload`), then `Restart backup timer`, so a changed schedule takes effect.

### What the script does

1. Stops the containers in `backup_stop_containers`, so their databases (SQLite for Pi-hole and NPM, BoltDB for Portainer) are copied in a consistent state. A `trap` on `EXIT` starts them again however the script ends, so a failed `tar` can't leave DNS down.
2. For each volume in `backup_volumes`, runs a throwaway `backup_helper_image` container that mounts the volume read-only and writes `<volume>_<YYYY-MM-DD_HHMM>.tar.gz` into `backup_dir`. Each archive is set to mode `0600`.
3. Deletes archives older than `backup_retention_days` days.

A run takes about 6 seconds on the test VM (2026-11-26), and that is how long DNS is down each night.

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `backup_dir` | `/var/backups/homelab` | Where the archives are written on the server. |
| `backup_retention_days` | `7` | Archives older than this are deleted (`find -mtime +7`, see Notes). |
| `backup_schedule` | `"*-*-* 02:30:00"` | `OnCalendar=` value for the timer. Keep it well clear of the 04:00 automatic reboot set by the `base` role. Test a new value with `systemd-analyze calendar "<value>"`. |
| `backup_helper_image` | `"alpine:3.24.4"` | Image that runs `tar`. Pinned; check [Alpine on Docker Hub](https://hub.docker.com/_/alpine) before bumping. |
| `backup_volumes` | `dns_pihole_data`, `proxy_npm_data`, `proxy_npm_letsencrypt`, `portainer_data` | Volumes to archive. Portainer prefixes each stack's volumes with the stack name, so these are the names `docker volume ls` shows, not the ones in the compose files. |
| `backup_stop_containers` | `pihole`, `npm`, `portainer` | Containers stopped during the backup. Unbound keeps no state and keeps running. |

## Example

```yaml
# playbooks/site.yml
- name: Configure homelab servers
  hosts: homelab
  become: true
  roles:
    - base
    - docker
    - dns_host
    - portainer
    - portainer_stacks
    - backup
```

Run a backup now, outside the schedule (before a risky change, for example):

```bash
ssh -t test-homelab 'sudo systemctl start homelab-backup.service'
```

Copy the newest archives to WSL, into `~/homelab-backups/<host>/`:

```bash
ansible-playbook playbooks/backup-pull.yml
```

## Checking it

On the server:

```bash
systemctl list-timers homelab-backup.timer     # NEXT and LAST run
journalctl -u homelab-backup.service -n 20     # the last run's output: "Backup finished: <stamp>"
sudo ls -l /var/backups/homelab                # the archives
```

## Notes

- **A backup on the server alone doesn't count.** It sits on the same disk as the data. Run `backup-pull.yml` to keep a copy on WSL. On the test VM both copies are still on the desktop's SSD, so the drill proves the process, not the 3-2-1 rule; in production the server and WSL are separate machines.
- **The archives are secrets.** They hold Pi-hole's password hash, NPM's admin account and the server certificate's private key. They're `0600` in a `0700` directory on the server, and in a `0700` directory on WSL, outside the repo. `.gitignore` also blocks `*.tar.gz` and `backups/`. Encrypt them before they go anywhere else, such as OneDrive.
- **Retention keeps 8–9 archives, not 7.** `find -mtime +7` rounds ages down to whole days, so it only matches files at least 8 full days old.
- **`Persistent=true`**: if the server was off at 02:30, the backup runs at the next boot.
- **RPO** (worst-case data loss) is about 24 hours with one nightly run. Changes made since the last run are lost in a restore, so run a backup by hand before a big change.
- Restoring: [`docs/runbooks/restore.md`](../../docs/runbooks/restore.md). Measured recovery times: [`docs/runbooks/disaster-recovery.md`](../../docs/runbooks/disaster-recovery.md).

## License

MIT
