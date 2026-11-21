# Week 7: Nginx Proxy Manager, local names and HTTPS

**Date:** 20–21 Nov 2026 · **Status:** done

## Goal

Reach `https://portainer.home.arpa`, `https://pihole.home.arpa` and `https://npm.home.arpa` with a padlock and no warning, using certificates from my own private CA. Close the old direct paths (Portainer on 9443, Pi-hole's web UI on 8081), and keep NPM's admin port off the LAN.

## Decided before starting

- **A private CA, not self-signed certificates or a real domain.** `home.arpa` (RFC 8375) can't get public certificates. A real domain with Let's Encrypt would need a DNS provider token on the server. Full reasoning in [ADR 0005](../adr/0005-private-ca.md).
- **The CA key stays outside Git, encrypted, with a Bitwarden backup** (ADR 0005, option C). Committing a Vault-encrypted key to a public repo was rejected: the ciphertext would be public forever, and rekeying doesn't clean Git history.
- **NPM is a Portainer Git stack** (`stacks/proxy/compose.yaml`), like the DNS stack. Ansible only creates the network it shares with Portainer.
- **Snapshots:** `07b-npm-deployed` after Step 3, and `07d-proxy-hosts` after the proxy hosts worked (NPM's settings aren't in Git, so the snapshot is their only backup on the test VM).

## What I did

Five steps, with small commits along the way because Portainer deploys stacks from GitHub:

| Commit | What |
| --- | --- |
| `e269f48` | Step 1: shared `proxy` network, Portainer on it, UFW rule so NPM can reach Pi-hole |
| `57f9d56` | Step 3: `stacks/proxy/compose.yaml` |
| `459713e` | Step 4: local DNS records in `stacks/dns/compose.yaml` |
| `c432cc5` | Step 5: drop the unused `host.docker.internal` mapping |
| milestone | `pki.yml`, the Step 5 role changes, ADR 0005 and these docs |

### Step 1: A network NPM shares with the services behind it

- **`docker` role:** a new last task creates the bridge network `proxy` (`docker_proxy_network`) with its subnet pinned to `172.20.0.0/24` (`docker_proxy_subnet`), clear of every subnet on the avoid list in [`ip-plan.md`](../network/ip-plan.md). In Step 5 its gateway was pinned too (`docker_proxy_gateway: "172.20.0.1"`).
- **`portainer` role:** Portainer joins `proxy` (`portainer_network`), so NPM can reach it as `portainer:9443`.
- **`dns_host` role:** Pi-hole is on the host network, so NPM has to reach it through the host. Entries in `dns_host_ufw_allowed` take an optional `from_ip` (default `any`), and a new entry allows `8081/tcp` from `dns_host_proxy_subnet`.

**The firewall puzzle.** A throwaway curl container on `proxy` tested the path to Pi-hole's web port:

| State of UFW | Result |
| --- | --- |
| `8081/tcp ALLOW Anywhere` | `302` (Pi-hole redirects to its login page) |
| Open rule deleted | `curl: (28) Connection timed out` |
| Only `8081/tcp ALLOW 172.20.0.0/24` | `302` |

`journalctl -k` showed the block: `[UFW BLOCK] IN=br-1093240cab01 SRC=172.20.0.3 DST=172.17.0.1 DPT=8081 … SYN`. The packet is addressed to the host itself, so it goes through UFW's INPUT rules, unlike a published container port, which Docker DNATs into the FORWARD chain ([Docker bypasses UFW](../security-baseline.md#docker-bypasses-ufw)). The other `[UFW BLOCK]` lines were IGMP queries from the BGW320 to `224.0.0.1`, harmless noise.

### Step 2: The private CA (`playbooks/pki.yml`)

The playbook runs on `localhost` only and uses `community.crypto` (pinned `>=3.5.1,<4.0.0` in `requirements.yml`). Everything goes to `~/.homelab-pki/` (`0700`), outside the repo.

| | Root CA "Homelab Root CA" | Server certificate for NPM |
| --- | --- | --- |
| Key | ECC P-384, PKCS#8, passphrase from Vault, `0600` | ECC P-256, no passphrase (NPM must read it), `0600` |
| Lifetime | 10 years (Nov 2036) | 397 days (until 23 Dec 2027) |
| Constraints | `CA:TRUE`, `pathlen:0`; `keyCertSign`, `cRLSign`; name constraint `DNS:home.arpa`, all critical | `CA:FALSE`; extended key usage `serverAuth` |
| Names | — | SANs `npm.home.arpa`, `pihole.home.arpa`, `portainer.home.arpa` |

- The passphrase is in `playbooks/vars/pki/vault.yml` (`vault_pki_ca_passphrase`), next to the playbook rather than in an inventory: only `pki.yml` loads it, and it isn't tied to one environment. The pre-commit hook checks it like every `vault.yml`.
- The play sets `ansible_python_interpreter: "{{ ansible_playbook_python }}"`. The inventory points every host, including the implicit `localhost`, at `/usr/bin/python3`, which doesn't have the `cryptography` library; Ansible's own Python (pipx) does.
- `ca.crt` is in Windows' Trusted Root store (`certmgr.msc`, current user), and the Firefox-based browser trusts Windows roots through `security.enterprise_roots.enabled`. The key and passphrase are backed up in a Bitwarden secure note.

### Step 3: Nginx Proxy Manager (`stacks/proxy/compose.yaml`)

| Setting | Value | Why |
| --- | --- | --- |
| Image | `jc21/nginx-proxy-manager:2.17.0@sha256:9127…3534` | Pinned by tag and digest |
| Ports | `80:80`, `443:443`, `127.0.0.1:81:81` | 80 and 443 for the LAN; the admin UI on loopback only |
| Volumes | `npm_data:/data`, `npm_letsencrypt:/etc/letsencrypt` | NPM's database, proxy hosts and uploaded certificates |
| Network | `proxy`, `external: true` | Created by Ansible, not by the stack |
| Restart | `unless-stopped` | Same policy as the other containers |

Deployed as Portainer Git stack `proxy`, with no environment variables. NPM 2.17 has no default login: the first visit to the admin UI shows a "create your admin account" page. I reached it through `ssh -N -L 8181:127.0.0.1:81 test-homelab` and `http://localhost:8181`. The account is in Bitwarden.

### Step 4: Local DNS names, as code

One line in `stacks/dns/compose.yaml`:

```yaml
      FTLCONF_dns_hosts: "${HOMELAB_IP:?Set HOMELAB_IP in the Portainer stack env vars} npm.home.arpa pihole.home.arpa portainer.home.arpa"
```

- `HOMELAB_IP` is a Portainer stack variable: `192.168.77.50` on the test VM, `192.168.77.10` in production. The IP doesn't belong in Git, because the same file runs in both places.
- `:?` makes the deploy fail with that message if the variable is missing. With a plain `${HOMELAB_IP}`, Compose would only warn and give Pi-hole a record with no IP.
- Pi-hole's format is `"IP HOSTNAME [HOSTNAME ...]"`, records separated by `;`. One record with three names is enough. Records set this way are locked in Pi-hole's UI.

### Step 5: HTTPS through NPM, then close the old paths

**Names on the desktop.** The desktop asks the production Pi-hole (.10) for DNS, which doesn't have these records yet. A temporary line in the Windows hosts file points the three names at `192.168.77.50`. It comes out in Week 10.

**Certificate and proxy hosts.** `server.crt` and `server.key` are uploaded as an NPM custom certificate, with no intermediate (the CA is a root). The file picker opened them straight from `\\wsl.localhost\…\.homelab-pki\`, so the key never sat in a Windows folder.

| Name | Forward to | Notes |
| --- | --- | --- |
| `npm.home.arpa` | `http://127.0.0.1:81` | NPM's own admin UI, inside its container |
| `pihole.home.arpa` | `http://172.20.0.1:8081` | The host's address on `proxy`; UFW allows only that subnet |
| `portainer.home.arpa` | `https://portainer:9443` | Container name on `proxy`; websockets on for Portainer's console |

All three use the custom certificate with Force SSL and HTTP/2. HSTS is off, because it would lock browsers out for months if the CA were ever rebuilt. Block Common Exploits and Cache Assets are off.

**Closing the old paths (Ansible):**

- `dns_host`: the old open 8081 entry stays in `dns_host_ufw_allowed` with `delete: true`, and the UFW task passes `delete: "{{ item.delete | default(false) }}"`. Deleting the entry from the list wouldn't have closed the port (the Week 3 lesson). One `ufw delete` removed both the IPv4 and IPv6 rules.
- `portainer`: `published_ports: []` with `comparisons: { published_ports: strict }`, and the unused `portainer_https_port` default removed. See finding 6 for why `strict` is needed.
- Portainer's first-run page is now reached through an SSH tunnel to the container's address, documented in the [`portainer` README](../../roles/portainer/README.md#notes).

## Checks

- [x] `dig @192.168.77.50 portainer.home.arpa +short` returns `192.168.77.50` (same for `npm` and `pihole`)
- [x] `https://portainer.home.arpa` and `https://pihole.home.arpa/admin` load with a padlock and no warning, and both log in; so does `https://npm.home.arpa`
- [x] `http://` versions redirect to `https://` (`301` from openresty)
- [x] From WSL, `curl -m 5 http://192.168.77.50:81` times out (admin panel not exposed)
- [x] `openssl s_client -connect 192.168.77.50:443 -servername portainer.home.arpa` shows `issuer=CN = Homelab Root CA`, and `Verify return code: 0 (ok)` with `-CAfile`
- [x] From WSL, `:8081` and `:9443` no longer answer; `docker port portainer` prints nothing
- [x] `sudo ufw status`: `8081/tcp` only from `172.20.0.0/24`, no `Anywhere` rule for it on IPv4 or IPv6
- [x] Second `site.yml` run: `changed=0`
- [x] `ansible-lint`: 0 failures, 0 warnings in 39 files, `production` profile

## What broke and how I fixed it

1. **The CA key came out in OpenSSL's old format.** Without `format:`, `openssl_privatekey` wrote `BEGIN EC PRIVATE KEY` with `Proc-Type: 4,ENCRYPTED`, which turns the passphrase into the AES key with one round of MD5. `format: pkcs8` switches to `BEGIN ENCRYPTED PRIVATE KEY`, with a salted, iterated PBKDF2. I deleted `~/.homelab-pki` and rebuilt the CA before anything trusted it.
2. **The encrypted CA key was pasted into a chat** while I was asking for help with the format. Encrypted or not, a key that has left the machine counts as exposed, so the rebuild above also replaced it. Rule since then: keys never get pasted anywhere.
3. **VS Code flagged `syntax-check[unknown-module]` for `community.crypto`.** The Ansible extension was linting with the system Python 3.12, which doesn't see the collection. `ansible-lint` in the terminal, the real check, was clean.
4. **HOMELAB_IP went into the wrong place at first.** My first draft added `HOMELAB_IP: "192.168.77.50"` to Pi-hole's `environment:` block, which only passes a variable into the container, where Pi-hole ignores it. Stack variables are substituted into `${...}` by Compose before the container starts; that's where the IP belongs.
5. **`pihole.home.arpa` gave `502 Bad Gateway`.** NPM's log for that host said `host.docker.internal could not be resolved (3: Host not found)`, yet `getent hosts host.docker.internal` inside the container returned `172.17.0.1`, and `curl` from inside NPM got `302` from Pi-hole. NPM builds `proxy_pass` from a variable, so nginx resolves the name when a request comes in, through its `resolver`: Docker's DNS at `127.0.0.11`. Docker's DNS knows container names (which is why `portainer` worked) but never reads `/etc/hosts`, where `extra_hosts` writes `host.docker.internal`. Fix: forward to `172.20.0.1`, the host's address on `proxy`, which comes from a subnet Ansible pins. The `extra_hosts` line was removed (`c432cc5`), and the gateway was pinned in the `docker` role; that task reported `ok`, because Docker had already picked `.1`.
6. **`published_ports: []` reported `ok` and left 9443 published.** By default, `docker_container` only checks that the ports you list are published, and an empty list always passes. With `comparisons: { published_ports: strict }` the next run recreated Portainer without the port, and the run after that was `ok`. Lesson: check the result from outside (`docker port`, `ufw status`, `curl`), not only Ansible's `changed`.
7. **NPM's admin account took about a minute to save.** NPM logged `Creating a new user in setup mode` at 10:45:17; the page just waited. The cause was the test environment, below.
8. **A manual "Pull and redeploy" restarted `unbound` as well as `pihole`,** although only Pi-hole's environment changed. In Week 6, a polling redeploy recreated only the changed container. In production that means a few seconds without DNS for the whole house.

### Test-environment limitation: VirtualBox's "green turtle"

The control node is WSL2 on Windows Home. WSL2 needs Windows' hypervisor, so VirtualBox runs on top of it (the green turtle icon) instead of using the CPU's virtualization directly. Symptoms on the test VM: soft lockups and RCU stalls in the kernel log, `journald`, `logind` and `udevd` failing to start after some boots, and slow saves like finding 7. Windows Home has no Hyper-V Manager, so the VM can't move to Hyper-V.

**Decision: keep WSL2 and live with it.** RAM wasn't the problem (21% used, no swap), and more vCPUs make the stalls worse. Turning off Hyper-V, VBS and Memory integrity, and going back to WSL1, was rejected. Habits instead: shut the VM down after each session, never let Windows sleep with it running, keep 2 vCPUs, use the Best performance power mode, and run `systemctl --failed` before trusting a test result. Production runs on bare metal, so this only affects testing.

## Evidence

Local DNS records (Step 4):

```text
$ dig @192.168.77.50 npm.home.arpa +short
192.168.77.50
$ dig @192.168.77.50 pihole.home.arpa +short
192.168.77.50
$ dig @192.168.77.50 portainer.home.arpa +short
192.168.77.50
$ ssh test-homelab 'docker exec pihole pihole-FTL --config dns.hosts'
[ 192.168.77.50 npm.home.arpa pihole.home.arpa portainer.home.arpa ]
```

The 502, and why (Step 5):

```text
$ ssh test-homelab 'docker exec npm sh -c "tail -n 5 /data/logs/proxy-host-*_error.log"'
==> /data/logs/proxy-host-2_error.log <==
[error] 302#302: *119 host.docker.internal could not be resolved (3: Host not found),
  client: 192.168.77.100, server: pihole.home.arpa, request: "GET /admin/ HTTP/2.0"

$ ssh test-homelab 'docker exec npm getent hosts host.docker.internal'
172.17.0.1      host.docker.internal
$ ssh test-homelab 'docker exec npm curl -sI -m 5 http://host.docker.internal:8081/admin/'
HTTP/1.1 302 Found
Location: /admin/login

$ ssh test-homelab "docker network inspect proxy --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'"
172.20.0.1
$ ssh test-homelab 'docker exec npm curl -sI -m 5 http://172.20.0.1:8081/admin/ | head -1'
HTTP/1.1 302 Found
```

HTTPS checks from WSL:

```text
$ curl -sI --resolve portainer.home.arpa:80:192.168.77.50 http://portainer.home.arpa | head -3
HTTP/1.1 301 Moved Permanently
Server: openresty

$ curl -m 5 http://192.168.77.50:81
curl: (28) Connection timed out after 5000 milliseconds

$ openssl s_client -connect 192.168.77.50:443 -servername portainer.home.arpa \
    -CAfile ~/.homelab-pki/ca.crt </dev/null 2>/dev/null | grep -E "issuer=|Verify return code"
issuer=CN = Homelab Root CA
Verify return code: 0 (ok)
```

After closing the old paths:

```text
TASK [portainer : Run the Portainer container]                changed
TASK [dns_host : Allow Pi-hole ports through UFW]
  ok      53/tcp, 53/udp, 8081/tcp from 172.20.0.0/24
  changed 8081/tcp delete: True      (first run)

$ ssh test-homelab 'docker port portainer'
(no output)
$ ssh -t test-homelab 'sudo ufw status'
22/tcp                     LIMIT       Anywhere                   # SSH
53/tcp                     ALLOW       Anywhere                   # Pi-hole DNS
53/udp                     ALLOW       Anywhere                   # Pi-hole DNS
8081/tcp                   ALLOW       172.20.0.0/24              # Pi-hole Web UI from NPM
22/tcp (v6)                LIMIT       Anywhere (v6)              # SSH
53/tcp (v6)                ALLOW       Anywhere (v6)              # Pi-hole DNS
53/udp (v6)                ALLOW       Anywhere (v6)              # Pi-hole DNS
$ curl -m 5 -sI http://192.168.77.50:8081/admin/
(no answer)
$ curl -m 5 -skI https://192.168.77.50:9443
(no answer)

test-homelab : ok=33  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
Passed: 0 failure(s), 0 warning(s) in 39 files processed of 67 encountered.
Last profile that met the validation criteria was 'production'.
```

## Notes for later weeks

- **Week 8:** the Portainer API work also creates the `proxy` stack, and the `dns` stack gets `HOMELAB_IP` along with `PIHOLE_PASSWORD`. NPM has an API too, which could create the three proxy hosts and close the gap in ADR 0005.
- **Week 9:** back up the `npm_data` volume and `~/.homelab-pki/`. The disaster-recovery drill should also try Portainer's first-run setup through the SSH tunnel, which is documented but hasn't been run on a fresh install yet.
- **Week 10 (production):**
  - Set `HOMELAB_IP=192.168.77.10` on the production `dns` stack, then remove the desktop's hosts-file line.
  - Recreate the proxy hosts in production's NPM, or restore `npm_data`.
  - Push DNS changes at quiet times: a redeploy can restart Unbound (finding 8).
- **Certificate renewal:** the server certificate expires on 23 Dec 2027. Re-run `pki.yml` and upload the new files to NPM before then.
- **Other devices:** phones and laptops will warn until `ca.crt` is imported on them.
- **NPM 2.17.1:** NPM's UI reports it, but GitHub still listed 2.17.0 as the latest release on 21 Nov. Read its notes before bumping the tag and digest.
- **Portainer 2.45.4:** bump `portainer_version` on its own day.
- **`base_ufw_allowed` can't delete rules yet.** `dns_host` now can; give `base` the same `delete` key when it's needed.
- **Still open from Week 4:** Docker's `live-restore`, before production.
