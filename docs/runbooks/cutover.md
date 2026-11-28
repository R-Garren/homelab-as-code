# Runbook: production cutover from v1 to v2

Week 10. Replaces the v1 server on the ThinkPad 11e with v2, built from this repo, and moves the household's DNS and DHCP back onto Pi-hole.

| | |
|---|---|
| **Goal** | The ThinkPad (192.168.77.10) runs v2: fresh Ubuntu Server 26.04, `site.yml`, the household's blocklists, NPM's proxy hosts. Pi-hole is the only DHCP server again |
| **Session length** | About 2 hours, plus the prep in Part 0 |
| **Where** | At the desk, with the desktop and the ThinkPad both in reach, the phone for the eero app, and the Ubuntu USB stick |
| **Household impact** | About an hour without ad-blocking; the internet stays up. Wi-Fi drops for a few minutes twice, when the eeros restart (Phases 1 and 7) |
| **Status** | Not run yet. Written 2026-11-27 |
| **Related** | [restore.md](restore.md) · [disaster-recovery.md](disaster-recovery.md) · [ip-plan.md](../network/ip-plan.md) · [migration-runbook.md](../network/migration-runbook.md) (same gateway and eero steps, used on 2026-11-17) |

Device MAC addresses and v1's DHCP settings are in `private/v1-dhcp-facts.md`, not in this file.

## How it stays safe

**One DHCP server at a time.** The AT&T gateway (BGW320) hands out addresses while the server is rebuilt, so the internet keeps working whatever happens to the ThinkPad. The order of the DHCP switches matters:

- **The gateway's DHCP goes on before v1 goes off** (Phase 1), and every device is made to reconnect. Devices then use the gateway as their DNS server, so they don't notice v1 shutting down.
- **The gateway's DHCP goes off just before `site.yml` runs** (Phase 4). Pi-hole's DHCP turns on by itself the moment Portainer deploys the `dns` stack: `PIHOLE_DHCP_ACTIVE: "true"` is set when the stack is created, and `portainer_stacks` only creates stacks, it never changes them. Turning the gateway off first means the two never run at the same time. The cost is 3–5 minutes with no DHCP server at all, which only affects a device joining in those minutes; devices with a lease keep it.
- **Only the desktop moves onto Pi-hole before the checks** (Phase 5). The gateway's leases last 1 hour, so everyone else stays on the gateway's DNS until Phase 7 makes them reconnect, after the checks have passed.

**The point of no return is the installer's "Confirm destructive action"** (Phase 3, step 3.4). Until then, v1 can be started again. After it, the fallback is the gateway's DHCP: working internet without ad-blocking, while v2 is fixed on the test VM. See [Rollback](#rollback).

---

## Part 0: Before the day

From the build guide's "Before Week 10: Pre-cutover prep", Part E. Tick these off before choosing the date.

- ☐ **Ubuntu Server 26.04.1 USB stick** written, and its SHA256 checksum matches the one on ubuntu.com.
- ☐ **ThinkPad BIOS checked:** USB boots first (or you know the one-time boot menu key), and Secure Boot is on, with Microsoft's UEFI CA allowed.
- ☐ **WSL `~/.ssh/config`** has a `Host homelab-prod` entry: `HostName 192.168.77.10`, `User homelab`, `IdentityFile ~/.ssh/homelab_v2`, `IdentitiesOnly yes`.
- ☐ **Fresh backup of the test VM, pulled to WSL.** Check that NPM's three proxy hosts are present on the VM first, then on the VM run `sudo systemctl start homelab-backup.service`, and in WSL run `ansible-playbook playbooks/backup-pull.yml`. Write its stamp here: `STAMP=__________`
- ☐ **v1 Teleporter export** in WSL `~/v1-export/` (Part A, 2026-11-27). Copy it to the desktop's Downloads folder too, because the import in Phase 5 happens in the browser.
- ☐ **Prod become password** in Bitwarden: the installer's password must be the same as `vault_ansible_become_password` in `inventories/prod/group_vars/all/vault.yml`.
- ☐ **`private/v1-dhcp-facts.md`** open: it has the five DHCP reservations to recreate in Phase 5.
- ☐ **Decided how Pi-hole's lists come across:** v1's lists through a Teleporter import (the default in Phase 5), or the test VM's lists by restoring `dns_pihole_data`. Write down which: `__________`
- ☐ **This runbook saved offline** on the phone. The internet may be patchy, and the server is down for part of the session.
- ☐ **Date set and household told:** about an hour without ad-blocking, the internet stays up, Wi-Fi drops for a few minutes twice. Anyone with trouble afterwards should turn Wi-Fi off and on again.

---

## Part 1: Pre-flight on the day (about 15 minutes)

**Don't start Phase 1 until every box is ticked.**

- ☐ In WSL, start the log. Every `mark` adds a timestamped line:
  ```bash
  mark() { echo "$(date +%T) $*" | tee -a ~/cutover-$(date +%F).log; }
  mark "START pre-flight"
  ```
- ☐ The `week-10` branch is merged and `main` is checked out and up to date (`git switch main && git pull`). Prod is built from `main`, because Portainer deploys the stacks from `main`.
- ☐ `ansible-lint` passes and CI is green on `main`.
- ☐ On v1 (`ssh homelab-v1`), note for the checks in Phase 5:
  - Pi-hole dashboard → **Domains on lists**: `__________`
  - One domain v1 blocks, from Query Log → blocked: `__________`
- ☐ Gateway (http://192.168.77.1) → Home Network → Subnets & DHCP: DHCP is **Off**, range 192.168.77.100–.249, lease **0 : 1 : 0 : 0** (1 hour). Fix the range and lease now if they differ, but leave DHCP **Off**.
- ☐ The gateway's Device Access Code is to hand (on the sticker).
- ☐ The test VM is shut down. It's not needed today.

---

## Phase 1: Hand DHCP to the gateway (about 15 minutes)

v1 keeps running in this phase, so its DNS still answers devices that haven't moved yet.

**1.1 Turn off v1's DHCP.**
v1's Pi-hole → Settings → DHCP → untick **DHCP server enabled** → **Save & Apply**.
- *Expected:* nothing visible. Devices keep their v1 leases.

**1.2 Turn on the gateway's DHCP.**
Gateway → Home Network → Subnets & DHCP → DHCP Server Enable **On** → **Save**. Check that the range is still .100–.249 and the lease is 1 hour.

**1.3 Make every device reconnect,** so each takes a gateway lease, with the gateway as its DNS server:
- **Wi-Fi:** eero app → Settings → Advanced networking → **Restart network**. Wi-Fi drops for 2–5 minutes.
- **Desktop:** `ipconfig /release`, then `ipconfig /renew` in Command Prompt.
- **PS5 and other wired devices:** restart them, or unplug and replug the cable.
- **Wait 5 minutes**, then check the gateway's Device → Device List.

**1.4 Phase 1 checks.**
- ☐ `ipconfig /all` on the desktop: 192.168.77.1xx, gateway **192.168.77.1**, DHCP server **192.168.77.1**, DNS **192.168.77.1**
- ☐ The phone (Wi-Fi) shows router 192.168.77.1 and loads websites
- ☐ v1's Pi-hole Query Log goes quiet: devices are asking the gateway now

`mark "Phase 1 done: gateway DHCP on"`

### 🚦 Gate 1

| Phase 1 checks pass? | Action |
|---|---|
| **Yes** | Continue to Phase 2 |
| **No**, after power-cycling and 10 more minutes | Gateway DHCP **Off**, v1's DHCP back **on**, eero **Restart network**. Stop for the day |

---

## Phase 2: Shut down v1 (5 minutes)

**2.1** In WSL: `ssh -t homelab-v1 'sudo poweroff'`. The `-t` gives `sudo` a terminal to ask for the password.

**2.2** Check: the phone and the desktop still load websites (through the gateway's DNS).

`mark "Phase 2 done: v1 off"`

v1 can still be started again from here: power on, then reverse Phase 1.

---

## Phase 3: Install Ubuntu Server (about 20–30 minutes, not counted in the RTO)

At the ThinkPad, boot the USB stick and install with these choices:

**3.1 Type of install:** Ubuntu Server (not the minimized version).

**3.2 Network:** choose the wired adapter (`enp5s0` on v1) → **Edit IPv4** → **Manual**:

| Field | Value |
|---|---|
| Subnet | 192.168.77.0/24 |
| Address | 192.168.77.10 |
| Gateway | 192.168.77.1 |
| Name servers | **192.168.77.1** (the gateway, not the server itself) |
| Search domains | leave empty |

The server must not use itself for DNS: Pi-hole doesn't exist yet when `site.yml` installs packages, and later the server would lose DNS every time Pi-hole restarts.

**3.3 Storage:** use the entire disk, **Set up this disk as an LVM group** ticked. On the summary screen, select `ubuntu-lv` → **Edit** → set the size to the maximum shown. (The installer's default uses only about half the disk; the test VM needed `lvextend` afterwards.)

**3.4 Confirm destructive action → Continue. ⚠️ This is the point of no return:** v1's disk is erased.

`mark "Phase 3: disk erase confirmed"`

**3.5 Profile:**

| Field | Value |
|---|---|
| Your name | homelab |
| Server name | homelab-prod |
| Username | homelab |
| Password | the prod become password from Bitwarden |

**3.6** Skip Ubuntu Pro. **Install OpenSSH server: on.** Allow password authentication over SSH: **on** for now, because `ssh-copy-id` needs it once; the `base` role turns it off. Import SSH key: no. Featured snaps: none.

**3.7** When it says **Reboot now**, remove the USB stick and reboot.

**3.8 At the ThinkPad's console,** log in as `homelab` and check the fresh install before Ansible touches it:
```bash
ip -br a                                        # enp5s0 has 192.168.77.10/24
df -h /                                         # close to the full disk, not about half
resolvectl status enp5s0                        # DNS server 192.168.77.1
timedatectl                                     # "System clock synchronized: yes"
systemctl --failed                              # 0 loaded units listed
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # write down this fingerprint
```
If `df` shows about half the disk: `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`.

`mark "Phase 3 done: Ubuntu installed"`

---

## Phase 4: Build v2 (about 5 minutes; the RTO clock starts here)

All from WSL, in the repo.

**4.1 Put the SSH key on the server.** v1's host key for this address is still in `known_hosts`, so remove it first:
```bash
ssh-keygen -R 192.168.77.10
ssh-copy-id -i ~/.ssh/homelab_v2.pub homelab@192.168.77.10
```
When it asks to accept the host key, compare the fingerprint with the one from 3.8 and type `yes` only if they match. Then check the key works:
```bash
ssh homelab-prod hostname        # homelab-prod, with no password prompt
```

**4.2 Dry run** (not counted in the RTO, the same as in the drill):
```bash
ansible-playbook -i inventories/prod playbooks/site.yml --check --diff
```
It fails partway, because later tasks need Docker, which a check run doesn't really install. Read the output for anything *unexpected*: the wrong host, a Vault error, a `become` error. A `Missing sudo password` or `Incorrect sudo password` means the installer's password doesn't match the vault.

**4.3 Start the clock, then turn off the gateway's DHCP.**
```bash
mark "RTO START"
```
Gateway → Subnets & DHCP → DHCP Server Enable **Off** → **Save**. From now on, devices keep their gateway leases (DNS 192.168.77.1) until Phase 7.

**4.4 Build:**
```bash
time ansible-playbook -i inventories/prod playbooks/site.yml
mark "site.yml done"
```
- *Expected:* `failed=0`, about 3 minutes (2 m 43 s on the test VM). Write down the `ok` and `changed` counts.
- **If it fails:** gateway DHCP back **On** first, then fix it. Running the playbook again is safe.

**4.5 Wait for the stacks.** Portainer deploys them in the background for a minute or two:
```bash
ssh homelab-prod "docker ps --format '{{.Names}}\t{{.Status}}'"
```
Wait until `pihole`, `unbound`, `npm` and `portainer` are **Up**, and `pihole` shows **(healthy)**.
```bash
dig @192.168.77.10 example.com +short     # answers
mark "containers up, DNS answering (empty lists)"
```

**4.6 Second run:**
```bash
ansible-playbook -i inventories/prod playbooks/site.yml
```
- *Expected:* `changed=0`.

---

## Phase 5: Bring the data across (about 10 minutes)

Three kinds of data come across, and two must **not**:

| Data | From | How |
|---|---|---|
| NPM proxy hosts and the certificate | Test VM backup | Volume restore (5.1) |
| Pi-hole lists, allow and deny entries, groups | v1 | Teleporter import (5.3) |
| DHCP reservations | `private/v1-dhcp-facts.md` | Typed in (5.4) |
| ~~`portainer_data`~~ | — | **Don't restore it from the test VM.** It holds the test VM's stack settings (`HOMELAB_IP` 192.168.77.50, DHCP off). Prod's Portainer was just built with the right ones |
| ~~v1's Pi-hole settings and leases~~ | — | Not imported: v2's settings come from the compose file, and v1's leases are stale |

**5.1 Restore NPM from the test VM's backup.**
This is the same command as [restore.md](restore.md) Part 1 step 4, with the archives from the test VM and the server set to prod:
```bash
HOST=homelab-prod
STAMP=__________                         # the test VM backup from Part 0
ssh "$HOST" 'mkdir -p -m 700 ~/restore'
scp ~/homelab-backups/test-homelab/proxy_npm_*_${STAMP}.tar.gz "$HOST":restore/
SRC='~/restore'
for pair in proxy_npm_data:npm proxy_npm_letsencrypt:npm; do
  VOLUME=${pair%%:*}; CONTAINER=${pair##*:}
  ssh "$HOST" "docker stop $CONTAINER && docker run --rm -v $VOLUME:/target -v $SRC:/restore:ro alpine:3.24.4 sh -c 'find /target -mindepth 1 -delete && tar -xzf /restore/${VOLUME}_${STAMP}.tar.gz -C /target' && docker start $CONTAINER"
done
ssh "$HOST" "docker ps --filter name=npm --format '{{.Names}}\t{{.Status}}'"
mark "NPM restored"
```
The proxy hosts point at container names and the Docker gateway (`172.20.0.1`), not at the test VM's address, so they work unchanged on prod.

**5.2 Move only the desktop onto Pi-hole.** This is the first real test of Pi-hole's DHCP, and it lets the desktop resolve the `home.arpa` names. Everyone else stays on the gateway until Phase 7.
1. The desktop's hosts file still sends the three names to the test VM. In Notepad run as administrator, open `C:\Windows\System32\drivers\etc\hosts`, delete the `192.168.77.50 … home.arpa` line and save.
2. In Command Prompt: `ipconfig /release`, `ipconfig /renew`, `ipconfig /flushdns`.
3. ☐ `ipconfig /all`: 192.168.77.1xx, gateway 192.168.77.1, **DHCP server 192.168.77.10**, **DNS 192.168.77.10** only, and Connection-specific DNS Suffix **home.arpa**.

`mark "desktop leased from Pi-hole"`

**5.3 Import v1's lists.**
Open `https://pihole.home.arpa/admin/` and log in with the prod Pi-hole password from Vault. (Port 8081 itself only accepts connections from NPM's Docker network, so go through the name.) Then Settings → Teleporter → **Import**, and choose the v1 export.

Before importing, **tick only the list items** (groups, lists, domains, clients). Untick the **configuration** (`pihole.toml`) and the **DHCP leases**. If the page names them differently, the rule is: lists yes, settings no, leases no.

Then Tools → **Update Gravity**, and check:
```bash
dig @192.168.77.10 <v1's blocked domain from Part 1> +short    # 0.0.0.0
```
- ☐ The dashboard's **Domains on lists** matches v1's number from Part 1.

`mark "RTO STOP: DNS answering with v1's lists"`

(If you chose the test VM's lists in Part 0, restore `dns_pihole_data` with 5.1's command instead of this step, with the pair `dns_pihole_data:pihole` and the `dns_pihole_data` archive copied over too.)

**5.4 Recreate the five DHCP reservations.**
Pi-hole → Settings → DHCP → **Static DHCP leases**. One line per device, in the form `MAC,IP,hostname`, from `private/v1-dhcp-facts.md` (the eeros at .2–.4, the test VM at .50, the printer at .52). **Save & Apply.**

---

## Phase 6: Checks (about 15 minutes)

All from the desktop, which is on Pi-hole since 5.2.

- ☐ **Week 6 checklist** against 192.168.77.10: `dig @192.168.77.10 example.com` answers; `dig @192.168.77.10 dnssec-failed.org` gives `status: SERVFAIL` (DNSSEC validation works); the blocked domain returns `0.0.0.0`
- ☐ **Week 7 checklist:** `nslookup npm.home.arpa` answers 192.168.77.10, and the same for `pihole` and `portainer`
- ☐ `https://npm.home.arpa`, `https://pihole.home.arpa` and `https://portainer.home.arpa` load with the private CA's certificate
- ☐ NPM shows the three proxy hosts with their original creation dates (they came from the backup)
- ☐ Portainer: log in as `admin`; stacks `dns` and `proxy` show **full control**; the footer says 2.45.4 LTS
- ☐ `ssh -t homelab-prod "sudo ufw status"` lists 53 (tcp and udp), 67/udp for DHCP, and 8081 only from 172.20.0.0/24
- ☐ `ssh homelab-prod "systemctl --failed"`: 0 units. `ssh homelab-prod "systemctl list-timers homelab-backup.timer"`: next run 02:30
- ☐ **Lid test:** close the ThinkPad's lid, wait a minute, then `ping -c 10 192.168.77.10` from WSL still answers
- ☐ `ssh -o PasswordAuthentication=yes -o PubkeyAuthentication=no homelab-prod` is refused: password login is off

`mark "Phase 6 checks done"`

### 🚦 Gate 2

| Phase 6 checks pass? | Action |
|---|---|
| **Yes** | Continue to Phase 7 |
| **A cosmetic problem** (a missing proxy host, a list count off by a little) | Note it, fix it after Phase 7 |
| **DNS, DHCP or the build is broken** and you can't fix it within 20 minutes | [Roll back](#rollback) to the gateway |

---

## Phase 7: Move the household onto Pi-hole (about 15 minutes)

**7.1 Make every other device reconnect,** as in 1.3: eero app **Restart network**; wired devices restarted. (The desktop moved in 5.2.) **Power-cycle the printer** so it picks up its .52 reservation.

**7.2 Checks after 5 minutes.**
- ☐ Pi-hole → Settings → DHCP → active leases: the eeros on .2–.4, the printer on .52, and the rest of the household filling in
- ☐ Pi-hole Query Log shows many individual clients
- ☐ The phone loads websites; ads are blocked on a test page
- ☐ The doorbell works, a TV streams, and a test page prints
- ☐ The phone shows router 192.168.77.1 and DNS 192.168.77.10

`mark "Phase 7 done: household on Pi-hole"`

**7.3 Tell the household it's done.**

---

## Rollback

What to do depends on how far you got:

| Reached | Rollback |
|---|---|
| Phase 1 or 2 | Power v1 on if it's off. Gateway DHCP **Off**, v1's DHCP back **on**, eero **Restart network**, desktop `ipconfig /renew`. Everything is back as it was |
| Phase 3 after 3.4, up to 4.3 | v1 is gone. Leave the gateway's DHCP **on** and stop: the household has working internet without ad-blocking. Fix the problem on the test VM, and continue from Phase 3 or 4 another day |
| 4.3 to Gate 2 | Stop Pi-hole first, so there is only ever one DHCP server: `ssh homelab-prod "docker stop pihole"`. Then gateway DHCP **On**, and `ipconfig /renew` on the desktop (it moved in 5.2). Everyone else is still on gateway leases, so nothing else changes. Fix it, then `docker start pihole`, gateway DHCP **Off**, and carry on from Phase 5 or 6 |
| After Phase 7 | The [Pi-hole is down](../network/ip-plan.md#runbooks) runbook: `docker stop pihole`, gateway DHCP **On**, eero **Restart network** |

Stopped containers stay stopped (`restart: unless-stopped`), and Portainer only redeploys a stack when `main` changes, so a stopped Pi-hole doesn't come back on its own.

---

## After the cutover

**The next 48 hours**
- ☐ Watch Pi-hole's Query Log and dashboard, and `ssh homelab-prod "docker ps"`, a few times a day.
- ☐ The morning after: the first nightly backup ran (`systemctl list-timers homelab-backup.timer` shows a `LAST` time of 02:30, and `/var/backups/homelab/` has four archives). Pull it with `ansible-playbook -i inventories/prod playbooks/backup-pull.yml`.
- ☐ The automatic reboot at 04:00 (when updates need one) didn't leave anything down.
- ☐ The gateway's Device List: no device still on a gateway lease. Power-cycle any straggler.

**Tidy up**
- ☐ Remove `Host homelab-v1` from WSL's `~/.ssh/config`. Retire the v1 key `ansible_homelab` (move it out of `~/.ssh/`, or delete it).
- ☐ Rebuild the test VM so its stacks follow `main`. Until now they track `refs/heads/week-10`, which the squash-merge deleted. Apply the `01-fresh-install` checkpoint, run `ansible-playbook playbooks/site.yml` without `-e portainer_stacks_git_ref` (the stacks then come from `main`), and restore `dns_pihole_data`, `proxy_npm_data` and `proxy_npm_letsencrypt` from the Part 0 backup ([restore.md](restore.md) Part 1). Don't restore `portainer_data`: it would put the stacks back on `week-10`. Check that Portainer shows `dns` and `proxy` on `refs/heads/main`, then take a new checkpoint.
- ☐ Write the measured RTO and the timeline from `~/cutover-<date>.log` into [disaster-recovery.md](disaster-recovery.md): install time, `site.yml` time, RTO, and what went differently from this runbook. Compare the RTO with the 10-minute target.
- ☐ Update this runbook's Status line, and fix any step that was wrong.

---

## Execution log

Copy the lines from `~/cutover-<date>.log` here. It becomes part of Week 10's build log.

| Time | Step | Result / notes |
|---|---|---|
| | | |

**Measured:** install ______ · `site.yml` ______ (`ok=__ changed=__`) · **RTO ______** (target 10 minutes)

**What went differently from this runbook:**
