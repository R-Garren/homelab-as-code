# Runbook: restore Docker volumes from a backup

How to put a backed-up Docker volume back on a server. Use it in two situations:

- **Undo a bad change** on a server that still works: restore one volume from last night's archive.
- **Disaster recovery**: after `site.yml` has rebuilt a server from zero, restore every volume (Part 2). The measured times are in [disaster-recovery.md](disaster-recovery.md).

| | |
|---|---|
| **Archives** | Made nightly at 02:30 by the `backup` role in `/var/backups/homelab/` on the server ([role README](../../roles/backup/README.md)). Copied to WSL by `playbooks/backup-pull.yml`, into `~/homelab-backups/<host>/` |
| **Downtime** | Each restored container is stopped for a few seconds. Restoring `dns_pihole_data` means a few seconds without DNS for the whole LAN |
| **Needs** | SSH to the server. The server user is in the `docker` group, so no `sudo` is needed: the restore runs in a throwaway container as root |
| **Status** | Part 1 tested 2026-11-26 on the test VM: `proxy_npm_data` restored in place, undoing a disabled proxy host; file owners unchanged. Part 2: tested in the DR drill on _date_ |

## What each archive holds

| Archive | Volume | Container to stop | Holds |
|---|---|---|---|
| `dns_pihole_data_<stamp>.tar.gz` | `dns_pihole_data` | `pihole` | Blocklists and allow/deny lists (`gravity.db`), settings (`pihole.toml`), query history, DHCP leases |
| `proxy_npm_data_<stamp>.tar.gz` | `proxy_npm_data` | `npm` | Proxy hosts, the uploaded custom certificate, NPM's admin account (`database.sqlite`) |
| `proxy_npm_letsencrypt_<stamp>.tar.gz` | `proxy_npm_letsencrypt` | `npm` | Empty: this lab uses a private CA, not Let's Encrypt. Restored for completeness |
| `portainer_data_<stamp>.tar.gz` | `portainer_data` | `portainer` | Portainer's database: users, settings, the Git source and the stacks |

`<stamp>` is the date and time of the run, for example `2026-11-26_1027`. Every archive from one run has the same stamp. **Restore archives from one run together**, never a mix of stamps.

**Portainer is optional.** Since Week 8, `site.yml` recreates Portainer's admin, Git source and stacks through the API ([ADR 0006](../adr/0006-portainer-admin-setup-token.md)). Restore `portainer_data` only if something was changed in Portainer by hand (a stack's environment variables, for example). The DR drill restores it anyway, to prove the archive works.

## Part 1: Restore one volume

Run from WSL. Replace the values in the first three lines.

### 1. Choose the archive

```bash
HOST=test-homelab                  # SSH alias of the server
STAMP=2026-11-26_1027              # from: ls ~/homelab-backups/$HOST/
VOLUME=proxy_npm_data              # see the table above
```

If the server still has its own archives, skip step 2 and set `SRC=/var/backups/homelab` in step 4 instead of `SRC=~/restore`.

### 2. Copy the archive to the server

```bash
ssh "$HOST" 'mkdir -p -m 700 ~/restore'
scp ~/homelab-backups/$HOST/${VOLUME}_${STAMP}.tar.gz "$HOST":restore/
```

`~/restore` is mode `0700` because the archives hold password hashes and private keys.

### 3. Record the current state (for comparison)

```bash
ssh "$HOST" "docker run --rm -v $VOLUME:/v:ro alpine:3.24.4 find /v -exec stat -c '%u:%g %n' {} +" | sort -k2 > /tmp/owners-before.txt
```

This lists every file in the volume with its numeric owner and group. After the restore, step 5 lists them again and compares (see Notes).

### 4. Restore

Stop the container that uses the volume, empty the volume, unpack the archive into it, start the container again. Use the container from the table (`npm` for `proxy_npm_data`):

```bash
CONTAINER=npm
SRC='~/restore'                    # or /var/backups/homelab, see step 1

ssh "$HOST" "docker stop $CONTAINER && \
  docker run --rm \
    -v $VOLUME:/target \
    -v $SRC:/restore:ro \
    alpine:3.24.4 \
    sh -c 'find /target -mindepth 1 -delete && tar -xzf /restore/${VOLUME}_${STAMP}.tar.gz -C /target' && \
  docker start $CONTAINER"
```

- `find /target -mindepth 1 -delete` empties the volume first, so files that weren't in the backup don't stay behind. It deletes the volume's contents, not the volume.
- If any step fails, `&&` stops the chain. **Check that the container is running again** (step 5) and run `docker start $CONTAINER` by hand if it isn't.
- To restore `dns_pihole_data`, use `CONTAINER=pihole`. DNS is down until it starts.

### 5. Check it worked

```bash
ssh "$HOST" docker ps --filter name=$CONTAINER        # Up, and (healthy) for pihole after about 30 s
ssh "$HOST" "docker run --rm -v $VOLUME:/v:ro alpine:3.24.4 find /v -exec stat -c '%u:%g %n' {} +" | sort -k2 > /tmp/owners-after.txt
diff /tmp/owners-before.txt /tmp/owners-after.txt    # see below
```

The `diff` should print nothing, or only lines for files that are newer than the backup (a log file created since, for example: a `<` line with no matching `>` line). **A file listed on both sides with a different owner is a problem**: see Notes.

Then the check for that service:

| Volume | Check |
|---|---|
| `dns_pihole_data` | `dig @<server IP> example.com +short` answers. A blocked domain returns `0.0.0.0`. The Pi-hole UI (`https://pihole.home.arpa`) shows the lists you expect under Lists |
| `proxy_npm_data` | `https://npm.home.arpa`, `https://pihole.home.arpa` and `https://portainer.home.arpa` load with the private CA's certificate. NPM's UI (tunnel: `ssh -N -L 8181:127.0.0.1:81 <host>`, then `http://localhost:8181`) shows every proxy host |
| `portainer_data` | `https://portainer.home.arpa`: log in as `admin`; both stacks show **Full control** |

### 6. Clean up

Once every check passes, delete the copies on the server. Until then, keep them: you may need to run step 4 again.

```bash
ssh "$HOST" 'rm ~/restore/*.tar.gz'
```

## Part 2: Restore everything after a rebuild from zero

Used in the disaster-recovery drill and in a real disaster. Start a timer at step 1: the time to working DNS is the RTO.

1. **Rebuild the server:** fresh Ubuntu Server 26.04 install (in the drill: restore the Hyper-V checkpoint `01-fresh-install`), then from WSL:
   ```bash
   ansible-playbook playbooks/site.yml
   ```
   Wait until `docker ps` shows `pihole`, `unbound`, `npm` and `portainer` running. The stacks deploy in the background for a minute or two after the playbook ends. The volumes exist now, but hold fresh, empty data.
2. **Copy all four archives** from one run to the server:
   ```bash
   HOST=test-homelab
   STAMP=2026-11-26_1027
   ssh "$HOST" 'mkdir -p -m 700 ~/restore'
   scp ~/homelab-backups/$HOST/*_${STAMP}.tar.gz "$HOST":restore/
   ```
3. **Restore the volumes**, each with Part 1 step 4, in this order:
   1. `dns_pihole_data` (container `pihole`): DNS comes back with your lists. **Stop the RTO timer** when `dig` answers and a blocked domain returns `0.0.0.0`.
   2. `proxy_npm_data` (container `npm`)
   3. `proxy_npm_letsencrypt` (container `npm`)
   4. `portainer_data` (container `portainer`), optional, see above
4. **Check everything:** the Week 6 and Week 7 checklists from the build guide, and every check in Part 1 step 5.
5. **Write down** in [disaster-recovery.md](disaster-recovery.md): the RTO, the stamp you restored and how old it was (the RPO in practice), and anything that didn't go as written here. Fix this runbook if a step was wrong.

## Notes

- **File owners.** The archives are made and unpacked by BusyBox `tar` in an Alpine container, while Pi-hole and NPM run on other images. Owners are stored as numbers, but a user name that exists in both images with a different number could be mapped wrongly. Step 3 and step 5 compare numeric owners for that reason. If they differ, add `--numeric-owner` to the `tar` command here and in the backup script.
- **Pi-hole's password** comes from the stack's `PIHOLE_PASSWORD` variable, not from the restored `pihole.toml`, so the password in Vault keeps working after a restore.
- **Mixing stamps:** restoring `proxy_npm_data` from one night and `proxy_npm_letsencrypt` from another is harmless today (the second is empty), but don't make it a habit.
- **Restoring on another server** (Week 10, test → production): the same steps work, because both servers get the same volume names from Portainer. Proxy hosts and the certificate come across; Pi-hole's DHCP leases from the test VM are meaningless in production, so restore `dns_pihole_data` there only if you want its lists and settings.
