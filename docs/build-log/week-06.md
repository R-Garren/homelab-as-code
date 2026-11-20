# Week 6: DNS stack (Pi-hole + Unbound from Git)

**Date:** 20 Nov 2026 · **Status:** done

## Goal

Run Pi-hole and Unbound as a Portainer Git-repository stack with full control, keep Unbound reachable only from the host itself, and prove that it validates DNSSEC.

## Decided before starting

- **Snapshot:** `06-before-dns` holds the Week 5 end state (Portainer running, admin account created).
- **Unbound image: `klutchell/unbound:1.26.2`,** pinned by tag and digest. v1's `mvance/unbound` hasn't been updated in over a year and every tag has the critical DNSSEC validator bug fixed in Unbound 1.26.1. Full comparison in [ADR 0004](../adr/0004-unbound-image.md).
- **The `dns_host` role owns Pi-hole's firewall rules,** not `base_ufw_allowed`. The guide suggested both. With the rules in the role, 67/udp can follow the DHCP variable (and be removed again with `delete:`), the ports live next to the service that needs them, and the production server gets them with the role in Week 10. `base_ufw_allowed` stays `[]`. The cost is that firewall rules now come from two roles, which the security baseline says.
- **DHCP is controlled by one variable per side:** `dns_host_dhcp_active` for the firewall rule, `PIHOLE_DHCP_ACTIVE` for Pi-hole itself. Both default to `false`, so forgetting to set them is safe. The guide called the variable `pihole_dhcp_active`, but ansible-lint wants a role's variables to start with the role's name.

## What I did

### Part A: Host preparation (role `dns_host`)

`roles/dns_host/` has `defaults/`, `handlers/`, `meta/`, `tasks/` and `templates/`, and is applied last in `site.yml`, after `base` (which turns UFW on) and `portainer`.

| Task | What it does |
| --- | --- |
| Make sure the resolved drop-in folder exists | `/etc/systemd/resolved.conf.d`, which doesn't exist on a fresh 26.04 install |
| Turn off the systemd-resolved stub listener | Templates `10-no-stub.conf` with `[Resolve]` / `DNSStubListener=no` and notifies `Restart systemd-resolved` |
| Point `/etc/resolv.conf` at the real upstream servers | Symlink to `/run/systemd/resolve/resolv.conf` |
| Create the Unbound config folder | `/opt/homelab/unbound`, `0755` |
| Template the Unbound override file | `pi-hole.conf`, `0644` root:root |
| Allow Pi-hole ports through UFW | 53/tcp, 53/udp and 8081/tcp from `dns_host_ufw_allowed` |
| Allow DHCP (67/udp) only when Pi-hole DHCP is active | `delete: "{{ not dns_host_dhcp_active }}"`, so turning DHCP off later also closes the port |

**Why `/etc/resolv.conf` breaks.** On Ubuntu it's a symlink to `../run/systemd/resolve/stub-resolv.conf` (the `--diff` output showed it), and that file lists only `127.0.0.53`, the stub listener this role turns off. `/run/systemd/resolve/resolv.conf` lists the real servers systemd-resolved got from DHCP instead. The handler only runs at the end of the play, so the symlink has already moved by the time the stub stops: the host never loses DNS during the run.

**Should the server use its own Pi-hole?** No. After the change, `dig example.com` on the VM is answered by `192.168.77.10`, the production Pi-hole, which the test VM gets from DHCP. If the host used its own Pi-hole, a stopped Pi-hole container would leave Docker unable to resolve Docker Hub, so it couldn't pull the image needed to fix it. In Week 10 the ThinkPad has a static address, so its DNS servers are set in netplan, and they mustn't be `127.0.0.1` or its own address.

**The Unbound override file.** `pi-hole.conf` takes Pi-hole's Unbound guide and changes four things for the container:

| Guide | Mine | Why |
| --- | --- | --- |
| `interface: 127.0.0.1` | left out (the image's `0.0.0.0` applies) | Inside a container, `127.0.0.1` is the container's own loopback; traffic from Docker's port mapping arrives on its network interface. |
| `port: 5335` | left out (the image's `53` applies) | The compose file maps `127.0.0.1:5335` to container port **53**. |
| `do-ip6: yes` | `no` | Docker networks are IPv4-only here, and the LAN has no global IPv6. Unbound would wait for IPv6 root servers to time out. |
| `root-hints:` | left out | The image ships its own root hints. |

Plus `access-control: 172.16.0.0/12 allow`, the whole range Docker hands out, because Pi-hole's queries come through Docker's port mapping from the stack network.

### Part B: `stacks/dns/compose.yaml`

| Service | Key settings |
| --- | --- |
| `unbound` | `klutchell/unbound:1.26.2@sha256:de8e…d8b4`; ports `127.0.0.1:5335:53` TCP and UDP; `pi-hole.conf` mounted read-only into `/etc/unbound/custom.conf.d/`; `restart: unless-stopped` |
| `pihole` | `pihole/pihole:2026.11.0`; `network_mode: host`; `cap_add: NET_ADMIN`; named volume `pihole_data` for `/etc/pihole`; `depends_on: unbound`; `restart: unless-stopped` |

Pi-hole environment:

| Variable | Value | Why |
| --- | --- | --- |
| `TZ` | `America/New_York` | Query log timestamps |
| `FTLCONF_webserver_api_password` | `${PIHOLE_PASSWORD}` | Typed into Portainer, never written in the repo |
| `FTLCONF_webserver_port` | `8081` | On the host network, Pi-hole's default is 80 and 443, which Nginx Proxy Manager needs in Week 7 |
| `FTLCONF_dns_upstreams` | `127.0.0.1#5335` | Unbound through its host-only port |
| `FTLCONF_dhcp_active` | `${PIHOLE_DHCP_ACTIVE:-false}` | Off unless Portainer sets the variable to `true` (production only) |
| `FTLCONF_ntp_ipv4_active`, `_ipv6_active`, `_sync_active` | `false` | Pi-hole v6 also runs an NTP server and client by default. The host already keeps its own time, so this means no time server on the LAN and no container trying to set the host clock |

Not set: `FTLCONF_dns_listeningMode`. Pi-hole's example sets it to `ALL`, but only because it uses Docker's bridge network. On the host network the default (answer only devices on local subnets) is the safer choice.

Before pushing, the file was checked without starting it: copied to the VM's `/tmp` and run through `PIHOLE_PASSWORD=dummy docker compose -f /tmp/compose.yaml config --quiet`, which printed nothing (valid). Starting it from a shell would have made Portainer show it as "limited" again, the Week 5 problem.

### Part C: Deploy it from Git through Portainer

**New in 2.45: Sources.** Portainer 2.45 keeps Git repositories as **Sources** under App Delivery, and a Git stack deploys from one. I added the source `homelab-as-code` (`https://github.com/R-Garren/homelab-as-code`, no authentication because the repo is public, access: Administrators), then **Stacks → Add stack → Repository** with:

- name `dns`;
- source `homelab-as-code`, reference `refs/heads/main`, compose path `stacks/dns/compose.yaml`;
- one environment variable, `PIHOLE_PASSWORD` (from Bitwarden). `PIHOLE_DHCP_ACTIVE` is deliberately not set.

The stack page shows **Managed by Git** with the repo, source, ref, file and commit, and no "control limited" warning. Its URL carries `id=2&type=2&regular=true`: Portainer has its own record of this stack ([screenshot](../img/week-06-stack-full-control.png)). That's the v1 → v2 payoff from ADR 0003.

On first start, Pi-hole built its blocklist (76,105 domains from the default StevenBlack list) and logged `7 FTLCONF environment variables found (7 used, 0 invalid, 0 ignored)`. Its web server came up on `0.0.0.0:8081` only, and `NTP sync is disabled`.

The 🔒 next to DHCP Settings in Pi-hole's web UI means the setting comes from an environment variable, so DHCP can't be switched on from the browser either.

**GitOps test.** I turned on polling for the source (every minute for the test), then added a label to the `unbound` service in Git and pushed (commit `1f89752`):

```yaml
    labels:
      homelab.deployed-by: "portainer-gitops"
```

Within about a minute Portainer showed commit `1f89752`, `unbound` had a new Created time (14:56:06), and `pihole` still showed 14:35:58: only the container whose settings changed was recreated ([screenshot](../img/week-06-gitops-redeploy.png)). On the VM:

```text
$ docker inspect unbound --format '{{json .Config.Labels}}' | grep -o '"homelab[^,]*'
"homelab.deployed-by":"portainer-gitops"
```

For normal use the source polls every 5 minutes.

## Checks

- [x] `dig @192.168.77.50 example.com` (from WSL): `NOERROR`
- [x] `dig @192.168.77.50 doubleclick.net`: `0.0.0.0`, blocked by Pi-hole (`EDE: 15 (Blocked): (gravity)`)
- [x] On the VM, `dig dnssec.works @127.0.0.1 -p 5335`: `NOERROR` with the `ad` flag
- [x] On the VM, `dig fail01.dnssec.works @127.0.0.1 -p 5335`: `SERVFAIL`
- [x] `dig @192.168.77.50 -p 5335 example.com` (from WSL): times out, Unbound isn't exposed to the LAN
- [x] `http://192.168.77.50:8081/admin`: the Pi-hole login page
- [x] Pi-hole → Settings → DHCP: off (and locked by the environment variable)
- [x] Pi-hole query log: the `dig` queries are listed, and the uncached one was forwarded upstream (Unbound at `127.0.0.1#5335` is the only upstream)
- [x] Portainer shows the stack as Managed by Git, with no "control limited" warning
- [x] A push to `main` is redeployed by Portainer on its own
- [x] Second `site.yml` run: `changed=0`
- [x] `ansible-lint`: 0 failures, 0 warnings in 36 files, `production` profile

## What broke and how I fixed it

1. **Mistakes caught in review, before any run.** The first drafts had: the web port set to `5353` (that's mDNS, which printers and Rokus use to find each other), a handler with `state: present` (not a valid service state), the stub-listener task using `file` with the Unbound config as its source, `state: present` on a directory, `owenr` for `owner`, the folder task pointing at `unbound.conf` instead of the folder, the config file mode `0755` (a config file shouldn't be executable), `access_control` with an underscore (Unbound would refuse to start), and `172.17.0.0/16`, then `172.17.0.0/12`, which isn't a valid start for a /12 (it's `172.16.0.0/12`). The pasted Pi-hole guide also brought `interface: 127.0.0.1`, `port: 5335` and `do-ip6: yes`, which would have broken Unbound in the container (Part A table).
2. **The dry run never reached the new role.** `dns_host` wasn't in `site.yml` yet; ansible-lint had still checked it, because it lints every role folder whether a playbook uses it or not.
3. **`Could not find or access '10-no-stub.conf.j2'`.** A typo in the template's file name. Ansible's "Searched in" list starts with where it expected the file (`roles/dns_host/templates/`). The real run had already created the empty drop-in folder before failing, which the next run simply reported as `ok`. No need to restore the snapshot: the tasks are idempotent.
4. **`yaml[trailing-spaces]` in `compose.yaml` line 5.** A blank line containing spaces. Removed.
5. **`ssh: connect to host 192.168.77.50 port 22: No route to host`.** The VM wasn't running at that moment; `uptime` later showed it had booted at about 14:25. "No route to host" means the machine isn't on the network at all, unlike "Connection refused" (nothing listening) or a timeout (a firewall dropping packets).
6. **The first `dig @192.168.77.50 example.com` from WSL timed out, and an SSH login hung.** Pi-hole logged `Long-term load (15min avg) larger than number of processors: 5.4 > 2`, and `uptime` showed a 5-minute load of 8.49 on 2 vCPUs: the VM had just booted, pulled two images and built the blocklist. With Unbound's cache empty, the first recursive lookup took longer than dig's 15 seconds. The TTLs proved Unbound had answered, just too late. Five minutes later everything answered in milliseconds. Testing each link separately found it: `docker exec unbound dig @127.0.0.1 example.com` (Unbound alone), `dig @127.0.0.1 -p 5335` (the port mapping), `dig @127.0.0.1` (Pi-hole to Unbound).
7. **`fail01.dnssec.works` timed out on the first try**, then dig's retry got `SERVFAIL` in 1 ms. When a signature fails, Unbound asks the zone's other servers before giving up, which took longer than dig's first 5-second wait; then it caches the bad result.
8. **The browser autofilled the Portainer admin password** into the Personal Access Token field of the new Source. Turning Authentication off removed the field, so it was never saved as a Git credential.
9. **The GitOps test didn't redeploy at first:** polling was still off on the source. After turning it on, the stack page showed `Polling: 1m` and the next poll picked up the commit.
10. **`git commit -am` would have made a broken commit.** It stages every tracked file that changed, so it would have committed `site.yml` with `dns_host` in it, but not the untracked role folder. The label commit used `git add stacks/dns/compose.yaml` and `git commit -m` instead.

## Evidence

Idempotency (second run):

```text
TASK [dns_host : Make sure the resolved drop-in folder exists]       ok
TASK [dns_host : Turn off the systemd-resolved stub listener]        ok
TASK [dns_host : Point /etc/resolv.conf at the real upstream servers] ok
TASK [dns_host : Create the Unbound config folder]                   ok
TASK [dns_host : Template the Unbound override file]                 ok
TASK [dns_host : Allow Pi-hole ports through UFW]                    ok (×3)
TASK [dns_host : Allow DHCP (67/udp) only when Pi-hole DHCP is active] ok

test-homelab : ok=32  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

Host after Part A:

```text
$ sudo ss -tulpn | grep ':53 '
(no output)
$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 32 Nov 20 14:06 /etc/resolv.conf -> /run/systemd/resolve/resolv.conf
$ dig example.com | grep -E 'status|SERVER'
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31731
;; SERVER: 192.168.77.10#53(192.168.77.10) (UDP)
$ sudo ufw status
22/tcp        LIMIT   Anywhere    # SSH
53/tcp        ALLOW   Anywhere    # Pi-hole DNS
53/udp        ALLOW   Anywhere    # Pi-hole DNS
8081/tcp      ALLOW   Anywhere    # Pi-hole Web UI
(plus the same four rules for IPv6)
$ ls -l /opt/homelab/unbound/
-rw-r--r-- 1 root root 2060 Nov 20 14:06 pi-hole.conf
```

Unbound config check:

```text
$ docker run --rm --entrypoint unbound-checkconf \
    -v /opt/homelab/unbound/pi-hole.conf:/etc/unbound/custom.conf.d/pi-hole.conf:ro \
    klutchell/unbound:1.26.2
Digest: sha256:de8e624f84337bdb2d16540c0b203d5df2a981180421cb50ab4212b67542d8b4
unbound-checkconf: no errors in /etc/unbound/unbound.conf
```

Containers:

```text
NAMES       IMAGE                      STATUS                PORTS
pihole      pihole/pihole:2026.11.0    Up (healthy)
unbound     klutchell/unbound:1.26.2   Up                    127.0.0.1:5335->53/tcp, 127.0.0.1:5335->53/udp
portainer   portainer/portainer-ce:2.45.3  Up                0.0.0.0:9443->9443/tcp
```

`unbound` has `172.18.0.2` on the stack network `dns_default`, which is why `access-control` allows `172.16.0.0/12` and not only Docker's default `172.17.0.0/16`.

DNSSEC, on the VM:

```text
$ dig dnssec.works @127.0.0.1 -p 5335
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37111
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; ANSWER SECTION:
dnssec.works.           3600    IN      A       46.23.92.212

$ dig fail01.dnssec.works @127.0.0.1 -p 5335
;; communications error to 127.0.0.1#5335: timed out
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 31501
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
```

The `ad` (authenticated data) flag means Unbound checked the signatures on `dnssec.works` and they're valid. `fail01.dnssec.works` has a deliberately broken signature, so Unbound refuses to answer: that's DNSSEC protecting the client from a forged answer.

From the LAN (WSL):

```text
$ dig @192.168.77.50 example.com
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5956
;; Query time: 4 msec
;; SERVER: 192.168.77.50#53(192.168.77.50) (UDP)

$ dig @192.168.77.50 doubleclick.net
; EDE: 15 (Blocked): (gravity)
doubleclick.net.        2       IN      A       0.0.0.0

$ dig @192.168.77.50 -p 5335 example.com
;; communications error to 192.168.77.50#5335: timed out
;; no servers could be reached
```

## Notes for later weeks

- **Week 7:** Pi-hole's web UI is plain HTTP on 8081, so its password crosses the LAN unencrypted. Put it behind Nginx Proxy Manager with the private CA. Pi-hole stays on the host network, so NPM reaches it on the host's address.
- **Week 8:** create the `homelab-as-code` source and the `dns` stack through Portainer's API, with `PIHOLE_PASSWORD` from Vault. Check how the 2.45 API handles Sources.
- **Week 10 (production):**
  - Set `dns_host_dhcp_active: true` in the production inventory and `PIHOLE_DHCP_ACTIVE=true` in the production stack.
  - Set the ThinkPad's own DNS in netplan to something other than itself.
  - Every push to `main` that changes a service restarts it, and a restarted Unbound means a few seconds without DNS for the whole house. Push DNS changes at quiet times, or point production at a release tag instead of `main`.
- **A cold cache after a reboot can time out the first lookups** (finding 6). That window is real after every 04:00 automatic reboot. Stretch goal: a healthcheck on `unbound` (the image ships `drill-hc`) with `depends_on: condition: service_healthy`.
- **A changed `pi-hole.conf` needs an Unbound restart.** Ansible updates the file, but Unbound only reads it on start.
- **Tidy-up:** remove the `so-rcvbuf` "expect a warning" comment from `pi-hole.conf.j2`; no such warning appeared.
- **Portainer 2.45.4 is out.** Bump `portainer_version` on its own day.
- **Still open from Week 4:** Docker's `live-restore`, before production.
