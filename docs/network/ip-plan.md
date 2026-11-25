# IP address plan: 192.168.77.0/24

**Status: in effect since 2026-11-17.** The home LAN moved from 192.168.1.0/24 to **192.168.77.0/24** in Week 0, Step 7. This is a plan, not an inventory: it lists address ranges and roles, not the devices or MAC addresses in the house. In later weeks it becomes Ansible variables.

## Why move off 192.168.1.0/24

192.168.1.0/24 is the most common home subnet. In Week 11 I'll connect to WireGuard from other people's networks, and many of them also use 192.168.1.x. When both ends use the same subnet, a phone can't tell which 192.168.1.10 is meant, and the tunnel breaks. An uncommon third octet avoids that.

The AT&T BGW320 reportedly accepts 192.168.x.x and 172.16.x.x LAN ranges but not 10.x.x.x, so the plan stays in 192.168.x.x.

## Network

| Setting | Value |
|---|---|
| Network | 192.168.77.0/24 |
| Subnet mask | 255.255.255.0 |
| Gateway | 192.168.77.1 (AT&T BGW320) |
| DNS | 192.168.77.10 (Pi-hole) only |
| DHCP server | Pi-hole (the BGW320's DHCP stays off) |
| Local domain | home.arpa (set in the v2 build; v1 keeps `lan` until then) |

## Address ranges

| Range | Use | How assigned |
|---|---|---|
| .1 | BGW320 gateway | Set on the gateway |
| .2–.9 | Network gear (the eero access points, once bridged) | Pi-hole DHCP reservations |
| .10–.49 | Servers and infrastructure | Static, set on the host itself |
| .50–.99 | Devices that need a fixed address | Pi-hole DHCP reservations |
| .100–.249 | Everything else | Pi-hole dynamic pool (150 addresses) |
| .250–.254 | Spare, for temporary testing | Manual |

**Golden rule:** a static or reserved address is never inside the dynamic pool.

## Assigned addresses

| Address | Role | How assigned |
|---|---|---|
| 192.168.77.1 | Gateway (BGW320) | Gateway setting |
| 192.168.77.2 | eero access point (gateway unit) | Reservation |
| 192.168.77.3 | eero access point | Reservation |
| 192.168.77.4 | eero access point | Reservation |
| 192.168.77.10 | ThinkPad 11e: Pi-hole, Unbound, Nginx Proxy Manager, Portainer | Static (netplan) |
| 192.168.77.50 | Ubuntu test VM (Hyper-V) | Reservation |
| 192.168.77.51 | Spare (was the OPNsense lab WAN, dropped from v2) | — |
| 192.168.77.52 | Printer | Reservation |

Phones, tablets and watches use private (randomized) MAC addresses. A reservation only works if the device's private address is set to **Fixed** for this network.

## DHCP scope (Pi-hole)

| Setting | Value |
|---|---|
| Range | 192.168.77.100–192.168.77.249 |
| Router (gateway) option | 192.168.77.1 |
| DNS server | 192.168.77.10 only |
| Lease time | 24 hours |
| Domain | home.arpa |

**Pool size check:** the Week 0 inventory counts about 25 dynamic devices (phones, laptops, tablets, streamers and smart devices), well under half of the 150-address pool. That leaves room for guests and growth.

**No public backup DNS server.** Devices pick between DNS servers unpredictably, so a public "backup" would let some lookups skip Pi-hole's blocking. If Pi-hole goes down, the fallback is to turn the BGW320's DHCP back on (see the runbook below).

## IPv6

LAN IPv6 is **off** on the BGW320 and on the eeros, and `ipconfig /all` shows no IPv6 DNS servers. That rules out devices resolving names through AT&T over IPv6 and bypassing Pi-hole. This is a known limitation, to revisit after v2.0.0.

## Other subnets to avoid

Any future subnet must not overlap these:

| Subnet | Used by |
|---|---|
| 192.168.77.0/24 | This home LAN |
| 172.30.64.0/20 | WSL on the control node (can change after a reboot) |
| 10.13.13.0/24 | WireGuard tunnel (Week 11) |
| 192.168.10.0/24 – 192.168.50.0/24 | Reserved for the future segmentation project (see [segmentation-plan.md](segmentation-plan.md)) |

## Migration

The change was done in one session on 2026-11-17, with both gates passed and no rollbacks, following [migration-runbook.md](migration-runbook.md): eeros to bridge mode first, then the re-address, with a go/no-go gate after each phase. The runbook has the exact steps, checks and rollback for each phase.

**Checks once both are done:**
- ☑ `ipconfig /all` shows a 192.168.77.x address, gateway 192.168.77.1 and DNS 192.168.77.10 only
- ☑ Pi-hole shows queries from individual clients
- ☑ `ssh <user>@192.168.77.10` reaches the server

## Availability risks

| Risk | When | Impact | How it's reduced |
|---|---|---|---|
| Devices keep their old 192.168.1.x address after the change | Re-address | A device that doesn't notice the change stays offline until its lease renews: 12–21 hours with 24-hour leases | Lease time deliberately kept at 1 day. Instead, every device is forced to reconnect: eero "Restart network" for Wi-Fi, `ipconfig /renew` on the desktop, power-cycling the console. The gateway's Device List shows any stragglers |
| Wi-Fi clients lose DNS (eero DNS hard-coded to 192.168.1.10) | Re-address, if eeros are still routing | 8 devices, including the video doorbell, lose name resolution | Eeros go to bridge mode **before** the re-address, which removes the eero DNS setting and the second NAT |
| Hard-coded old addresses on the server | Re-address | v1 services behind Nginx Proxy Manager, local DNS records and VPN configs break | Every hit from `grep -rn "192\.168\.1\."` is listed with its new value before starting |
| Port forwards on the gateway still point at 192.168.1.x | Re-address | Inbound services (for example, the VPN) stop working | Firewall → NAT/Gaming checked and screenshotted beforehand, then updated |
| Two DHCP servers on at once, or none | DHCP handover | Conflicting addresses, or new devices can't join | The handover is one action. The gateway's temporary DHCP uses 1-hour leases so its clients move to Pi-hole quickly |
| Server unreachable after a netplan typo | Re-address | No DNS or DHCP for the house | `sudo netplan try` reverts on its own after 120 seconds. The change is made at the server's keyboard |
| Wi-Fi drops while the eeros restart | Bridge mode | A few minutes without Wi-Fi | Done at a quiet time with the household warned |
| Weaker signal after the gateway's Wi-Fi is turned off | Bridge mode | About 10 devices move from the gateway's radio to the eeros. Some may get a weaker signal | Signal checked in the eero app afterwards. Stragglers power-cycled, doorbell first |
| Guide and runbook unavailable during the outage | Both | No instructions while the internet is down | Runbook and guide kept offline: PDF or a OneDrive file marked "available offline" |
| **Pi-hole becomes a single point of failure** | Permanent, after both changes | One server provides DHCP and DNS for every device. If it fails, DNS stops at once and new devices can't join | The "Pi-hole is down" runbook below. The laptop's battery rides out short power cuts. Pi-hole config is backed up with Teleporter. A second DNS server is a later improvement |

## Runbooks

**Rollback (Step 7 failed):** set the BGW320 back to 192.168.1.254 with its DHCP on, restore the old netplan file on the ThinkPad and run `sudo netplan try`.

**Pi-hole is down:** turn on the BGW320's DHCP (range .100–.249). Devices get internet with the gateway's DNS until Pi-hole is back, then turn the BGW320's DHCP off again.
