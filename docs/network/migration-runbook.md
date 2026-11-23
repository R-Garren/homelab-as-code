# Migration runbook: bridge the eeros and re-address to 192.168.77.0/24 (one session)

Week 0, Steps 7 and 8, done back to back in a single session.

| | |
|---|---|
| **Goal** | One flat network (192.168.77.0/24), one NAT and firewall (the AT&T gateway), one DHCP and DNS server (Pi-hole), one Wi-Fi network (the eeros) |
| **Session length** | About 2–2.5 hours, plus prep the day before |
| **Where** | At the desk with the desktop (gateway port 1) and the ThinkPad 11e (gateway port 3) both in reach, with a phone for the eero app |
| **Household impact** | Wi-Fi drops for a few minutes twice; internet and DNS are patchy for up to about 1 hour of the session |
| **Related** | [ip-plan.md](ip-plan.md) (targets and availability risks) · [current-topology.md](current-topology.md) (before state) |

**Order changed from the build guide:** the eeros go to bridge mode (Step 8) **first**, then the LAN is re-addressed (Step 7). Each phase ends with a go/no-go gate, so a problem is caught while only one change is in play:

- **Bridge mode runs on the old, known-good subnet**, so if it breaks, I know which change caused it.
- **It removes two Step 7 risks** before Step 7 starts: the eeros' DNS setting hard-coded to 192.168.1.10, and their separate 192.168.4.x network.
- **It makes every device visible to Pi-hole**, so the eero, printer and console MAC addresses are in Pi-hole's lease list by the time their reservations are created.

Device MAC addresses are in `private/inventory.md`, not in this file.

---

## Part 0: The day before

**Status: complete (2026-11-17).** Two items are deferred to after the session: the eero firmware update and the DHCP reservations. Both are on the "next day" list in Part 3.

- **Pi-hole's lease time stays at 1 day** (decided 2026-11-16). That makes step 2.10, forcing every device to reconnect, essential. A device that doesn't reconnect can keep its old 192.168.1.x address for up to about 21 hours.
- ⏭ **Update the eero firmware:** deferred until after the migration (decided 2026-11-17). Not needed for this session. Never do it on the day itself.
- ☑ **Pi-hole Teleporter backup exported** (2026-11-17) and saved on the desktop, not on the server. This is the Phase 2 rollback.
- ☑ **Gateway port forwards checked** (Firewall → NAT/Gaming, screenshot in `private/evidence/`). There is **one rule**: "WireGuard", UDP 51820 → the server (listed by its network card maker, "Quanta Computer Inc."). The gateway ties the rule to the device, not to a typed IP address.
- ☑ **v1 WireGuard forward deleted** (2026-11-16). Nobody uses the v1 VPN, and Week 11 recreates it with fresh keys. The custom service definition "*WireGuard" is still saved on the gateway and can be reused then. Step 2.7 is skipped.
- ☑ **Hard-coded address list built** (2026-11-17, `grep` of `/etc` and both compose files):

  | Where | What | Handled in |
  |---|---|---|
  | `/etc/netplan/00-installer-config.yaml` | address, `via`, first nameserver | Step 2.5 |
  | `/opt/homelab/core/docker-compose.yml` | Portainer (9000, 9443) and NPM (81, 443) ports bound to `192.168.1.10`; `FTLCONF_LOCAL_IPV4` | Step 2.6 (one `sed`). **Without this, Portainer and NPM won't start** |
  | Same file, `DHCP_*` lines | Old v5 variable names. Pi-hole v6 ignores them (the DHCP page is editable) | Nothing to do, Pi-hole's UI controls DHCP |
  | Pi-hole local DNS records | `pihole.lan`, `portainer.lan` → 192.168.1.10; `wazuh.lan` (no Wazuh container is running) | Step 2.6 |
  | Nginx Proxy Manager proxy hosts | `pihole.lan` → `https://192.168.1.10:8443`; `wazuh.lan` → `https://192.168.1.10:5601` (Wazuh not running). `portainer.lan` → `http://portainer:9000` uses the container name, no change | Step 2.6 |
  | `/etc/iptables/rules.v4` | 4 leftover v1 rules redirecting port 53 to 192.168.1.10. Would have broken DNS after the switch | ☑ **Removed** 2026-11-17 (live and in the file); DNS tested OK |
  | v1 WireGuard container | Not needed until Week 11 | ☑ **Stopped** 2026-11-17 (`docker compose down`) |
  | Unbound | Listens on 127.0.0.1 only | Nothing to do |
  | Conditional forwarding | Not configured | Nothing to do |

  Ignore the hits in `udevil.conf` and `openssl.cnf` (example comments) and `resolv.conf.bak` (old backup).
- ☑ **Compose file backed up:** `/opt/homelab/core/docker-compose.yml.pre-77` (the Phase 2 rollback uses it).
- ☑ **Server's network config backed up** (2026-11-17). This is the rollback once `netplan try` has been confirmed:
  ```bash
  sudo cp /etc/netplan/00-installer-config.yaml ~/00-installer-config.yaml.bak
  ```
- ☑ **Nothing will overwrite the netplan file at reboot** (2026-11-17): `/etc/cloud/cloud.cfg.d/00-subiquity-disable-cloudinit-networking.cfg` has `network: {config: disabled}`.
- ⏭ **DHCP reservations:** deferred (2026-11-17). Pi-hole has none today, and nothing in this session depends on them: the server's address is set in netplan, and every other device takes a normal address from the pool. Add them after the session from Pi-hole's lease list (Part 3, "The next day").
- ☑ **Pi-hole's current DHCP range** on 192.168.1.0/24 is .64–.253 (190 addresses), plenty for Phase 1, when every Wi-Fi device starts leasing from Pi-hole. Checked from the backup.
- ☑ **Runbook saved offline on the phone** (2026-11-17). Re-sync it after any edit to this file.
- ☑ **Household told** (2026-11-17).

---

## Part 1: Pre-flight (about 10 minutes)

Do this at the start of the session. **Don't start Phase 1 until every box is ticked.**

- ☐ Pi-hole Settings → DHCP: DHCP server **enabled**, lease time **1 day**
- ☐ Gateway (http://192.168.1.254) → Home Network → Subnets & DHCP: DHCP server **Off**
- ☐ The Teleporter backup file is on the desktop
- ☐ `~/00-installer-config.yaml.bak` exists on the server (`ls ~/*.bak`)
- ☐ `/opt/homelab/core/docker-compose.yml.pre-77` exists on the server
- ☐ This runbook is open on the desktop (the hard-coded address list is in Part 0)
- ☐ The gateway's Device Access Code is to hand (it's on the sticker)
- ☐ The phone has the eero app, cellular data and the offline runbook
- ☐ Write the start time in the execution log at the bottom of this file

---

## Phase 1: eeros to bridge mode (about 30–40 minutes)

Everything stays on 192.168.1.0/24 in this phase.

**1.1 Switch the eeros to bridge mode.**
eero app → Settings → Advanced networking → DHCP & NAT → **Bridge** → ✓ (confirm the warning).
- *Expected:* the eeros restart and Wi-Fi drops for 2–5 minutes.

**1.2 Confirm a Wi-Fi client is on the main LAN.**
On the phone: Settings → Wi-Fi → (i) next to the network.
- *Expected:* a 192.168.1.x address and router **192.168.1.254**. A 192.168.4.x address means bridge mode isn't active yet: wait 2 minutes and check again.
- In Pi-hole → Query Log, the phone now appears under its own IP, not 192.168.1.71.

**1.3 Turn off the gateway's own Wi-Fi.**
Gateway → Home Network → Wi-Fi. Set the 2.4 GHz and 5 GHz networks to **Off** (the "Wi-Fi Operation" or "Network" setting for each radio), then **Save**.
- *Expected:* the roughly 10 devices on the gateway's radio drop and rejoin through the eeros, because the network name and password match.

**1.4 Wait 10 minutes, then account for every device.**
Compare Pi-hole → Settings → DHCP (active leases) with `private/inventory.md`. Power-cycle anything missing, in this order:
1. The doorbell (security device, so first)
2. TVs and streamers
3. Smart speakers
4. Everything else

**1.5 Check signal strength.**
eero app → Devices → each TV or streamer that used to be on the gateway's radio. Note any with weak signal (one or two bars) for a later fix, such as moving an eero.

**1.6 Run the Phase 1 checks.**
- ☐ `tracert 1.1.1.1` from a laptop on Wi-Fi shows **one** private hop (192.168.1.254)
- ☐ The desktop prints a test page (the printer is no longer behind a second NAT)
- ☐ Doorbell live view works; one TV streams
- ☐ Pi-hole's lease list includes the three eeros, the printer and the PS5. (Reservations are deferred, so there's no need to copy their MACs today.)

**1.7 Take "after" screenshots for Phase 1:** eero DHCP & NAT showing Bridge, and Pi-hole's lease list.

### 🚦 Gate 1: go or no-go

| All Phase 1 checks pass? | Action |
|---|---|
| **Yes** | Write the time in the log and continue to Phase 2 |
| **No**, after power-cycling and 10 more minutes | **Roll back Phase 1 and stop.** Don't start the re-address on a broken base |

**Rollback Phase 1:**
1. eero app → DHCP & NAT → **Automatic**.
2. Gateway → Wi-Fi → both radios back **On** → Save.
3. Devices return to the old 192.168.4.x and 192.168.1.x networks within a few minutes.

---

## Phase 2: Re-address to 192.168.77.0/24 (about 60–90 minutes)

**Do this phase at the ThinkPad's keyboard.** SSH to the server drops at step 2.5.

**2.1 Turn off Pi-hole's DHCP.**
Pi-hole → Settings → DHCP → untick **DHCP server enabled** → **Save & Apply**.
- *Expected:* nothing visible. Clients keep their current leases (up to a day) until step 2.10 forces them to reconnect.

**2.2 Re-address the gateway and turn on its temporary DHCP.**
Gateway → Home Network → Subnets & DHCP. Set all of these, then **Save** once:

| Field                | Value                                             |
| -------------------- | ------------------------------------------------- |
| Device IPv4 Address  | 192.168.77.1                                      |
| Subnet Mask          | 255.255.255.0                                     |
| DHCP Server Enable   | **On**                                            |
| DHCPv4 Start Address | 192.168.77.100                                    |
| DHCPv4 End Address   | 192.168.77.249                                    |
| DHCP Lease           | **0 : 1 : 0 : 0** (1 hour, not the default 1 day) |

- *Expected:* the page stops responding. That's normal, because the gateway's address just changed.
- **Why the 1-hour lease:** devices that join during the next half hour get the gateway as their DNS server, which bypasses Pi-hole. With a 1-day lease they would stay that way for most of a day.

**2.3 Move the desktop to the new subnet.**
In Command Prompt:
```
ipconfig /release
ipconfig /renew
ipconfig /all
```
- *Expected:* a 192.168.77.1xx address and gateway 192.168.77.1.
- Log in at **http://192.168.77.1**. Check that Subnets & DHCP shows the new values and Broadband status is Up.
- **If you can't reach 192.168.77.1 within 10 minutes:** unplug and replug the desktop's cable and renew again. If that still fails, go to the Phase 2 rollback.

**2.4 Note who's on the gateway's temporary DHCP.**
Gateway → Device → Device List. Devices on 192.168.77.x are working. Those still on 192.168.1.x haven't renewed yet, which is expected; step 2.10 handles them.

**2.5 Re-address the server.** At the ThinkPad, edit the live file:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
Change exactly three values. Retype only the numbers and leave the indentation alone: YAML breaks on tabs or shifted spaces.

| Line | From | To |
|---|---|---|
| `addresses:` under `enp5s0` | `192.168.1.10/24` | `192.168.77.10/24` |
| `via:` | `192.168.1.254` | `192.168.77.1` |
| first `nameservers` address | `192.168.1.254` | `192.168.77.1` |

Leave `8.8.8.8`, `dhcp6` and the MAC match as they are. Save with **Ctrl+O, Enter**, then exit with **Ctrl+X**. Then apply:
```bash
sudo netplan try
```
If the file has a syntax error, `netplan try` refuses it and nothing changes: reopen it in nano and fix it.

While the 120-second timer runs, open a second terminal (Ctrl+Alt+F2, or a new tab) and run:
```bash
ip -br a                 # shows 192.168.77.10/24
ping -c3 192.168.77.1    # gateway answers
ping -c3 1.1.1.1         # internet answers
```
- **All three pass:** go back and press **Enter** to keep the new config. You'll see `Configuration accepted.` From this point the change is permanent: the file in `/etc/netplan/` is what the server reads at every boot. Then confirm it saved:
  ```bash
  sudo grep -rn "192\.168" /etc/netplan/   # only 192.168.77.x values
  ls /etc/netplan/                          # only 00-installer-config.yaml
  resolvectl status enp5s0                  # DNS servers 192.168.77.1 and 8.8.8.8
  ```
  Netplan applies every `.yaml` in that folder, so any other file there could bring the old address back at boot.
- **Any fail:** do nothing. Netplan reverts to the previous config by itself after 120 seconds. Check your three edits and try again. **After two failed attempts, roll back Phase 2.**

**2.6 Bring the services back on the new address.**
Portainer and NPM are tied to the old IP in the compose file, so they stop answering as soon as the server's address changes. At the ThinkPad:
```bash
sudo sed -i 's/192\.168\.1\.10/192.168.77.10/g' /opt/homelab/core/docker-compose.yml
sudo docker compose -f /opt/homelab/core/docker-compose.yml up -d
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```
- *Expected:* pihole, unbound, nginx-proxy-manager and portainer are all **Up**. NPM and Portainer show `192.168.77.10:…` ports. Pi-hole gets recreated too, so DNS drops for a few seconds.

From the desktop, these pages load (use the IP, not `.lan` names, yet):
- Pi-hole: `http://192.168.77.10/admin`
- Portainer: `https://192.168.77.10:9443`
- NPM: `http://192.168.77.10:81`

Then fix the names:
- **Pi-hole → Settings → Local DNS Records:** change `pihole.lan` and `portainer.lan` to **192.168.77.10**. Delete `wazuh.lan`.
- **NPM → Proxy Hosts:** edit `pihole.lan` (⋮ → Edit) and change the forward IP to **192.168.77.10**, keeping port 8443 and https. Delete `wazuh.lan`. Leave `portainer.lan` alone.

**2.7 Port forwards: nothing to do.** The only rule (v1 WireGuard) was deleted before the migration. Glance at Firewall → NAT/Gaming to confirm it still says "No Application Hosting entries have been defined".

**2.8 Set Pi-hole's new DHCP scope, but don't enable it yet.**
Pi-hole → Settings → DHCP:

| Setting             | Value                           |
| ------------------- | ------------------------------- |
| DHCP server enabled | leave **off** for now           |
| Range               | 192.168.77.100 – 192.168.77.249 |
| Router              | 192.168.77.1                    |
| Netmask             | 255.255.255.0                   |
| Lease time          | **1d**                          |

**Skip the reservations today** (deferred in Part 0). Every device takes a normal address from the pool. The table below is for the "next day" task in Part 3:

| IP | Device |
|---|---|
| 192.168.77.2 | eero, gateway unit |
| 192.168.77.3 | eero |
| 192.168.77.4 | eero |
| 192.168.77.50 | Ubuntu test VM |
| 192.168.77.52 | printer |
| 192.168.77.53 | PS5 |
| 192.168.77.54 | Nintendo Switch |

Then **Save & Apply** (with DHCP still **off**).

> **Leave the local domain as it is** (`lan`) in this session. Moving to `home.arpa` happens in the v2 build. Changing it now would break `pihole.lan`-style names for no benefit today.

**2.9 Swap the DHCP servers, within one minute.**
1. Gateway → Home Network → Subnets & DHCP → DHCP Server Enable **Off** → **Save**.
2. **Immediately:** Pi-hole → Settings → DHCP → tick **DHCP server enabled** → **Save & Apply**.

**2.10 Force every device onto a Pi-hole lease.** With 1-day leases, this step is what moves devices onto the new subnet. Don't skip any part of it.
- **Wi-Fi:** eero app → Settings → Advanced networking → **Restart network**. If the option isn't there in bridge mode, unplug the gateway eero's power for 10 seconds. Every Wi-Fi client reconnects and asks Pi-hole for a lease.
- **Desktop:** `ipconfig /release` then `ipconfig /renew`.
- **PS5:** restart it, or unplug and replug its cable.
- **ThinkPad:** nothing to do, its address is static.
- **Wait 5 minutes,** then power-cycle any device that's missing from Pi-hole's lease list (doorbell first).

**2.11 Run the Phase 2 checks.**
- ☐ `ipconfig /all` on the desktop: 192.168.77.x, gateway **192.168.77.1**, DHCP server **192.168.77.10**, DNS **192.168.77.10** only
- ☐ Pi-hole → Query Log shows queries from many individual 192.168.77.x clients
- ☐ `ssh <user>@192.168.77.10` from WSL on the desktop works (it asks once to accept the key for the new address; type `yes`)
- ☐ `tracert 1.1.1.1` shows **one** private hop: 192.168.77.1
- ☐ Pi-hole's lease list shows the eeros, the printer and the consoles with 192.168.77.1xx addresses. Devices that were asleep get theirs when they wake
- ☐ The doorbell works, a test page prints, a TV streams, and a phone loads websites. If printing fails, the desktop probably added the printer by its old IP: remove it in Windows and add it again
- ☐ `sudo docker ps` on the ThinkPad: pihole, unbound, nginx-proxy-manager and portainer all **Up** (wireguard is stopped on purpose)
- ☐ `nslookup pihole.lan` on the desktop answers **192.168.77.10**
- ☐ `https://pihole.lan` opens the Pi-hole admin page and `https://portainer.lan` opens Portainer (both through NPM)
- ☐ No device in the gateway's Device List still shows a 192.168.1.x address as "on"

### 🚦 Gate 2: go or no-go

| All Phase 2 checks pass? | Action |
|---|---|
| **Yes** | Write the time in the log and continue to Part 3 |
| **A few devices are missing** | Power-cycle them. A device that won't come back on its own is a note for the log, not a reason to roll back |
| **Core checks fail** (desktop has no DNS, SSH fails, Pi-hole, Portainer or NPM won't stay up) and you can't fix them within 20 minutes | **Roll back Phase 2** |

**Rollback Phase 2.** This returns to the verified Phase 1 state: eeros stay bridged, on 192.168.1.0/24.
1. Pi-hole → Settings → DHCP → untick **DHCP server enabled** → Save & Apply.
2. Gateway (http://192.168.77.1) → Subnets & DHCP → Device IPv4 Address **192.168.1.254**, DHCP **On**, range 192.168.1.100–.249 → Save.
3. At the ThinkPad, restore the old network config:
   ```bash
   sudo cp ~/00-installer-config.yaml.bak /etc/netplan/00-installer-config.yaml
   sudo netplan try
   ```
   Confirm with Enter once `ping -c3 192.168.1.254` works. Then put the old compose file back (skip this if you never reached step 2.6):
   ```bash
   sudo cp /opt/homelab/core/docker-compose.yml.pre-77 /opt/homelab/core/docker-compose.yml
   sudo docker compose -f /opt/homelab/core/docker-compose.yml up -d
   ```
4. Pi-hole → Settings → Teleporter → **Import** the backup from Part 0. That restores the old DHCP scope (enabled), local DNS records and settings.
   If you reached step 2.6, also open NPM at `http://192.168.1.10:81` → Proxy Hosts → `pihole.lan` → set the forward IP back to **192.168.1.10**. (`wazuh.lan` can stay deleted.)
5. Gateway → Subnets & DHCP → DHCP **Off** → Save. Pi-hole is the only DHCP server again.
6. Port forwards: nothing to restore (there are none).
7. eero app → **Restart network**; on the desktop, `ipconfig /release` and `/renew`.
8. Check: the desktop is on 192.168.1.x with DNS 192.168.1.10, and SSH works. Write down what failed and stop for the day.

---

## Part 3: Wrap-up (about 20 minutes, same session)

- ☑ **Reboot test.** This proves the new address and the services survive a restart or power cut. The house loses DNS for about 2 minutes. At the ThinkPad:
  ```bash
  sudo reboot
  # after it comes back:
  ip -br a          # enp5s0 shows 192.168.77.10/24
  sudo docker ps    # pihole, unbound, nginx-proxy-manager, portainer all Up
  ```
  Then from the desktop: `nslookup google.com 192.168.77.10` answers, and `https://pihole.lan` opens.
  - **If Portainer or NPM isn't up:** run `sudo docker compose -f /opt/homelab/core/docker-compose.yml up -d`. Their ports are tied to the server's IP, and once in a while Docker starts before the network is ready. Note it in the log.
  - **If the server comes back on 192.168.1.10:** something in `/etc/netplan/` still has the old address. Rerun the checks at the end of step 2.5.
- ☑ **Export a new Pi-hole Teleporter backup.** Keep the pre-change one too.
- ☑ **Take "after" screenshots:** the gateway's Subnets & DHCP, Pi-hole DHCP settings and leases, eero DHCP & NAT, and `ipconfig /all`.
- ☐ **Save the new `ipconfig /all` and `tracert` output** to `private/evidence/`.
- ☐ **Update the inventory:** each device's new address, and any device that needed manual help.
- ☐ **Update the desktop** wherever it still uses 192.168.1.10: `~/.ssh/config` in WSL, the v1 Ansible inventory and browser bookmarks.
- ☐ **Fill in the execution log below.**
- ☐ **Tell the household it's done.** Anyone whose device misbehaves should turn Wi-Fi off and back on.

## The next day

- ☐ **Check Pi-hole's lease list** against the inventory once more, and the gateway's Device List for anything still on 192.168.1.x. Power-cycle any straggler. With 1-day leases, a device that was asleep or missed the reconnect only shows up here.
- ☐ **Add the DHCP reservations** deferred from Part 0, using the table in step 2.8. Copy each MAC from Pi-hole's lease list (Settings → DHCP), then restart that device so it picks up its reserved address. The test VM's MAC is in VirtualBox (VM → Settings → Network → Adapter 1 → Advanced).
- ☐ **Update the eero firmware** (deferred from Part 0), on a different day from any other change. Confirm all three eeros come back online.
- ☐ **Add the "After" diagram** to [current-topology.md](current-topology.md).
- ☐ **Show someone else in the house the "Pi-hole is down" fallback** from [ip-plan.md](ip-plan.md): gateway → Subnets & DHCP → DHCP **On**.

---

## Execution log

Fill this in during the session. It becomes the build-log entry for Week 0.

| Time | Step | Result / notes |
|---|---|---|
| 2026-11-17 07:55 | Part 0 done | Leftover iptables DNS rules removed, WireGuard stopped. eero firmware and reservations deferred |
| before 11:58 | Part 1 pre-flight done | All boxes ticked |
| before 11:58 | 1.1 eeros to bridge | No problems |
| before 11:58 | 1.3 gateway Wi-Fi off | No problems |
| by 11:58 | Gate 1 | **Go.** Pi-hole lease list (screenshot 11:58): all 25 devices on 192.168.1.x straight from Pi-hole, none on 192.168.4.x; eeros, printer and doorbell present |
| by 12:02 | 2.2 gateway re-addressed | 192.168.77.1/24, temporary DHCP .100–.249 with 1 h lease (screenshot 12:02) |
| 12:02–12:12 | 2.5 server re-addressed | `netplan try` accepted, 192.168.77.10/24 |
| by 12:12 | 2.6 services back up on .77.10 | Local DNS records updated, NPM `pihole.lan` → 192.168.77.10:8443, `wazuh.lan` deleted (screenshot 12:12) |
| by 12:17 | 2.9 DHCP swapped | Desktop leased 192.168.77.100 from Pi-hole (.77.10) at 12:17, gateway .77.1, DNS .77.10 only |
| 12:27 | Gate 2 | **Go.** All 2.11 checks passed; SSH to 192.168.77.10 OK (12:19); gateway DHCP confirmed **Off** (screenshot 12:27) |
| ~12:33 | Reboot test | **Pass.** Server came back on 192.168.77.10; all four containers Up and healthy without manual help |
| 12:37 | Part 3 done | New Teleporter backup exported 12:34 (kept next to the pre-change one). After-evidence saved, including an nmap DHCP discover showing Pi-hole as the only DHCP server. Old server keys removed from the desktop's PowerShell `known_hosts`. Still open: inventory update, WSL SSH and `~/.ssh/config`, telling the household |

**Devices that needed manual help:** None so far.

**What went differently from this runbook:**
- Nothing went wrong. Everything ran as written, with no rollbacks.
- The step 2.11 SSH check was run from PowerShell instead of WSL. Still to do: SSH in once from WSL (it keeps its own `known_hosts`), and remove the old entries with `ssh-keygen -R 192.168.1.10` and `ssh-keygen -R 192.168.1.213`.
- Spotted for later (no action today): the server picked up two IPv6 ULA addresses (`fda5:…`, `fdeb:…`) even though the gateway's IPv6 is off, so some device on the LAN is sending router advertisements. The server's clock is on UTC. 88 apt updates and 3 firmware updates are waiting (do these on a separate day).
