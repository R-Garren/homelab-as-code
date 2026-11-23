# Segmentation plan (target design)

**Status: design only, not part of v2.** The AT&T BGW320 has no VLAN support and the eero mesh puts all Wi-Fi in one VLAN, so this can't be enforced on today's hardware. It was planned as Week 11 of v2, an OPNsense lab in VirtualBox, but was dropped on 2026-11-23 (see Decisions). It stays here as the design for a standalone project, in Cisco Packet Tracer or on real hardware. Running it for real would need a VLAN-capable router, a managed switch and VLAN-aware access points.

Today's network is described in [current-topology.md](current-topology.md). The home LAN plan is in [ip-plan.md](ip-plan.md).

## Zones

| VLAN | Name | Subnet | Gateway | Who lives there |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | Router, switch and access points (3 mesh APs today) |
| 20 | Servers | 192.168.20.0/24 | 192.168.20.1 | ThinkPad 11e (Pi-hole, Unbound, Nginx Proxy Manager, Portainer); Ubuntu test VM |
| 30 | Trusted | 192.168.30.0/24 | 192.168.30.1 | Admin desktop, laptop, 7 phones, 2 tablets, a watch, a handheld gaming PC |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | 5 streaming players or TVs with Roku, a Google TV, 3 smart speakers or streamers, a video doorbell, 2 game consoles, a printer |
| 50 | Guest | 192.168.50.0/24 | 192.168.50.1 | Visitors' devices |

**Why these subnets:** the third octet matches the VLAN ID, so an address tells you its zone at a glance. None of them overlap the networks already in use: the home LAN (192.168.77.0/24), VirtualBox host-only (192.168.56.0/24), WSL (172.30.64.0/20) and the planned WireGuard tunnel (10.13.13.0/24).

**Inside each /24**, the layout copies the home LAN plan: .1 gateway, .10–.49 static, .50–.99 reservations, .100–.249 DHCP pool.

**Placement rules:**
- Anything you can't patch or audit yourself goes in IoT, even if trusted people use it (TVs, consoles, the printer).
- A general-purpose computer you control goes in Trusted (the handheld gaming PC runs desktop Linux, so it's a laptop, not a console).
- **Unknown devices start in IoT** and only move once identified.
- The admin desktop stays in Trusted. Its extra access comes from firewall rules (below), not from a zone of its own.

**One Trusted zone** (decided 2026-11-16). Splitting it further, for example a separate admin or kids zone, would add rules to maintain without protecting much more on a home network this size.

## Firewall matrix

Rows are the **source**, columns the **destination**. Everything not listed is denied. The firewall is stateful, so replies to an allowed connection are always let back in. "Deny" only stops a zone *starting* a connection.

| From ↓ / To → | Management | Servers | Trusted | IoT | Guest | Internet |
|---|---|---|---|---|---|---|
| **Management** | allow | DNS only | deny | deny | deny | allow (firmware updates) |
| **Servers** | deny | allow | deny | deny | deny | allow |
| **Trusted** | admin only ¹ | DNS + HTTPS; SSH for admin ¹ | allow | limited ² | deny | allow |
| **IoT** | deny | DNS only | deny | allow ³ | deny | allow |
| **Guest** | deny | DNS only ⁴ | deny | deny | deny ³ | allow |
| **Internet** | deny | WireGuard only ⁵ | deny | deny | deny | — |

1. **Admin desktop only.** It gets HTTPS and SSH to Management and SSH to Servers. It's also the Ansible control node, so it needs SSH to reach the servers. Every other Trusted device gets DNS (53) and HTTPS (443) to Servers only, which covers the web UIs behind Nginx Proxy Manager.
2. **Printing and casting only, to specific devices.** Trusted → printer on IPP (631) and raw printing (9100). Trusted → TVs and streamers on the ports their casting apps use. Nothing else.
3. **Traffic inside one VLAN never reaches the firewall,** so it can't be filtered there. IoT devices can still talk to each other. Guests should be kept apart with **client isolation** on the guest Wi-Fi network instead.
4. **Guests use Pi-hole for DNS** (decided 2026-11-16), so visitors get ad and tracker blocking too. The rule allows port 53 to Pi-hole and nothing else in Servers, so guests can't reach the Pi-hole web UI or any other service.
5. **v2 Week 11:** UDP 51820 forwarded to the WireGuard container (v2.1.0). No other inbound ports.

## DNS enforcement

Pi-hole only works if devices actually use it. Many TVs and streamers have Google DNS (8.8.8.8) built in and ignore what DHCP tells them.

- **Every zone's DHCP**, Guest included, hands out Pi-hole (192.168.20.10) as the only DNS server.
- **Outbound DNS (port 53) from any zone except Servers** is redirected to Pi-hole with a NAT rule, so hard-coded DNS servers get answered by Pi-hole anyway.
- **Outbound DNS over TLS (853)** is blocked except from Servers, where Unbound needs it if you add it later.
- **DNS over HTTPS** runs on port 443 and can't be blocked by port. A Pi-hole blocklist of known DoH servers is the practical fix. Document it as a known gap.

## Services that cross zones

- **mDNS repeater between Trusted and IoT only.** Phones find printers (AirPrint) and TVs (casting) through mDNS, which stays inside one VLAN on its own. Repeating it into Guest or Management would leak device names.
- **NTP:** every zone can reach the firewall for time. It's harmless, and it keeps logs accurate.
- **Consoles:** online multiplayer may want UPnP or specific port forwards. If so, allow it for the IoT zone only. Never move consoles into Trusted to fix a game.

## Wi-Fi mapping

| SSID | VLAN | Notes |
|---|---|---|
| Home (main) | 30 Trusted | WPA3 or WPA2/WPA3 mixed |
| IoT | 40 IoT | WPA2, 2.4 GHz allowed (many smart devices only support it) |
| Guest | 50 Guest | Client isolation on; a separate password that's easy to change |
| none | 10 Management | Management is wired only |

The current eeros can't map SSIDs to VLANs. This table is for the lab design and for future access points.

## Lab checks

Test every cell of the matrix from a device in each zone, and record the result next to the expected one:

- ☐ `ping` and `nmap -sn` from IoT to Trusted fail
- ☐ `dig @192.168.20.10` works from IoT, and `curl https://192.168.20.10` fails
- ☐ `dig @8.8.8.8 example.com` from IoT is answered by Pi-hole (redirect works)
- ☐ SSH to Servers works from the admin desktop and fails from another Trusted VM
- ☐ Guest reaches the internet and nothing internal
- ☐ The firewall's log shows the denied attempts

## Decisions

| Date | Decision | Why |
|---|---|---|
| 2026-11-16 | One Trusted zone | Extra zones add rules without much extra protection at this size |
| 2026-11-16 | Guests use Pi-hole for DNS (port 53 only) | Visitors get ad and tracker blocking; nothing else in Servers is reachable |
| 2026-11-16 | Unidentified Apple device confirmed as a phone → Trusted | Its Siri and Music lookups looked like a HomePod's, which would have meant IoT |
| 2026-11-23 | Removed from v2 (was Week 11); kept as a standalone project | Out of scope for the rebuild, and a multi-VM OPNsense lab runs poorly in VirtualBox on this desktop. Cisco Packet Tracer or real hardware suits it better |

## Still open

- ☐ Exact casting ports for the TVs and streamers (note 2): find them during the lab
