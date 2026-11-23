# Week 8: Automate Portainer through its API

**Date:** 23 Nov 2026 · **Status:** done

## Goal

One `ansible-playbook playbooks/site.yml` takes a bare VM to a running DNS and proxy stack with no clicks in Portainer. Ansible creates Portainer's admin account and tells Portainer which Git stacks to deploy. Portainer still owns the stacks, so they stay fully editable in its UI.

**Result:** from `01-fresh-install` to running stacks in **5 m 02 s**. The only manual step left is NPM's proxy hosts, about 3 minutes.

## Decided before starting

- **Explore by hand first.** Every call the role makes was tried with `curl` and `jq` from WSL, and its status code written down before any Ansible was written.
- **The admin is created through the API with a custom setup token from Vault** ([ADR 0006](../adr/0006-portainer-admin-setup-token.md)). Rejected: `--admin-password-file`, which needs the password in plain text on the server forever and stops Portainer from starting if the file goes missing, and `--no-setup-token`.
- **The API calls run on the server, against the Portainer container's address on the `proxy` network.** From WSL, Portainer is only reachable through NPM, and on a fresh server NPM has no proxy hosts yet. The cost is `validate_certs: false`, because Portainer's own certificate is self-signed; the traffic never leaves the server.
- **Stacks are data, not tasks:** `portainer_stacks_list` in `group_vars`, one loop in the role.
- **Snapshots:** `08-before-api` at the start, `08b-api-role` before the from-zero test, `08c-from-zero` after it passed.

## What I did

One milestone commit this week: the compose files the stacks use were already on `main`, so nothing had to be pushed early.

### Step 1: Explore the API with curl

WSL doesn't read the Windows hosts file, so a small shell function sent every request to the test VM by name, with the private CA for TLS:

```bash
pcurl() { curl -sS --resolve portainer.home.arpa:443:192.168.77.50 --cacert ~/.homelab-pki/ca.crt "$@"; }
P=https://portainer.home.arpa
```

- `--resolve` pins the name to the VM's address and keeps TLS name checking on, so `-k` was never needed.
- The password went in with `read -rs` and `jq -n --arg`, so it never touched the shell history or a quoted string. The JWT stayed in a shell variable; only its length was printed.

| Request | Status | What I learned |
| --- | --- | --- |
| `GET /api/system/status` | 200 | Version `2.45.3`; no login needed |
| `GET /api/users/admin/check` | 204 | An admin exists (404 on a fresh install) |
| `POST /api/auth` | 200 | Returns `{"jwt": …}`, used as `Authorization: Bearer <jwt>` |
| `GET /api/endpoints` | 200 | The environment's ID is **3**, not 1: it came from the browser setup |
| `GET /api/stacks` | 200 | `dns` and `proxy`; the `dns` stack shows `GitSourceId: 1` |
| `POST /api/stacks/create/standalone/repository?endpointId=1` | 404 | "Unable to find an environment"; wrong ID |
| same, `endpointId=3` | 200 | Throwaway stack `scratch-api-test` (a `hello-world` compose file on branch `api-test`) created as stack 4 |
| same POST again | 409 | "A stack with the normalized name … already exists" |
| `GET /api/stacks/4` | 200 | `GitSourceId: 1`, polling `5m` |
| `GET /api/gitops/sources` | 200 | One source, `homelab-as-code`, interval `5m` |
| `DELETE /api/stacks/4?endpointId=3` | 204 | Cleaned up; branch `api-test` deleted too |

Things the online examples got wrong for 2.45, confirmed in Portainer's source code at tag `2.45.3`:

- **Stacks point at a Git source.** The create body takes `SourceID`; `RepositoryURL` still works but is deprecated. When only a URL is sent, Portainer finds or creates a source with that URL, which is why the scratch stack ended up on source 1.
- **Polling lives on the source,** not on the stack: `AutoUpdate.Interval` is deprecated.
- **The create response leaves out `GitSourceId`.** It showed `null` even though the stack was linked; a `GET` on the stack shows it.
- **`admin/init` needs the setup token in an `X-Setup-Token` header.** It returns 403 without it and 409 if an admin already exists. `PORTAINER_SETUP_TOKEN` sets a custom token, and Portainer ignores it once an admin exists.
- **Without `-H`, Portainer 2.45 never creates a Docker environment.** In Weeks 5–7 the browser's "Get started" step did that. With `-H unix:///var/run/docker.sock`, Portainer creates one named `primary` right after `admin/init`. This would have broken the from-zero test, so it was fixed before Step 3.
- **Field names differ in case:** stacks and environments use `Id` and `Name`; sources use `id` and `name`.

### Step 2: Ansible changes

**New Vault entries** in `inventories/test/group_vars/all/vault.yml`: `vault_portainer_setup_token` (from `openssl rand -hex 32`), `vault_portainer_admin_password` and `vault_pihole_password`. The two passwords were already in Bitwarden, and the token went in next to them. `vars.yml` maps them to role variables, as it does for the become password.

**`portainer` role:**

- `env: PORTAINER_SETUP_TOKEN: "{{ portainer_setup_token }}"` (default `""`, which means Portainer's own random token).
- `command: ["-H", "{{ portainer_docker_endpoint }}"]` (default `unix:///var/run/docker.sock`).
- `no_log: true` on the container task, because the module prints its environment with `-v`.
- On the existing VM, each change recreated the container once (`changed`), and the next run was `ok`. The admin, the stacks and the existing `local` environment all survived on the `portainer_data` volume.

**New role `portainer_stacks`** ([README](../../roles/portainer_stacks/README.md)), in this order:

1. Find the container's IP on `proxy` and wait for `GET /system/status` to return 200.
2. `GET /users/admin/check` with `status_code: [204, 404]`; `POST /users/admin/init` only `when` it returned 404.
3. `POST /auth`, then keep `Authorization: Bearer <jwt>` in a fact.
4. Wait for the environment whose URL is `unix:///var/run/docker.sock` and save its ID. Looking it up by URL finds both `local` (this VM) and `primary` (a fresh one).
5. Create the source `homelab-as-code` (interval `5m`) if no source has that name, and save its ID.
6. List the stacks and create each item of `portainer_stacks_list` whose name is missing, with `SourceID`, `refs/heads/main`, the compose path and its `env`. `HOMELAB_IP` comes from the inventory's `ansible_host`, so production gets `.10` without another variable.

Rules followed:

- `changed_when` set by hand: the creating POSTs report `changed`, while the GETs and the login never do.
- `no_log: true` on every task that carries the password, the token or the JWT.
- `default([], true)` on every list, because Go sends an empty list as `null`.

**`playbooks/site.yml`** now runs `base`, `docker`, `dns_host`, `portainer`, `portainer_stacks`. `dns_host` moved ahead of `portainer` (finding 9).

**`docker` and `dns_host`** each end with `meta: flush_handlers` (finding 8).

On the existing VM: lint passed at the `production` profile, a run showed `changed=0` with all three create steps skipped, and `-vvv` output contained no secret.

### Step 3: The real test, from zero

Restored `01-fresh-install` and ran `time ansible-playbook playbooks/site.yml` without touching Portainer's UI.

- **First run:** `changed=28`, `failed=0`, `real 5m2.177s`. The handlers ran mid-play, right after `docker` and right after `dns_host`. "Create the admin user", "Create the Git source" and both stack items reported `changed`.
- **Second run:** `changed=0`. Every create step was skipped.
- **About two minutes later,** `npm`, `pihole` (healthy), `unbound` and `portainer` were all up.
- **NPM by hand,** about 3 minutes, the same steps as Week 7: the admin account through the tunnel, the custom certificate, and the three proxy hosts.
- **In Portainer** (environment `primary`, ID 1), both stacks were Managed by Git, on source `homelab-as-code` with `5m` polling, owned by administrators, with Stop, Delete, Edit stack settings, Pull and redeploy and Detach from Git: full control.

It took three from-zero attempts: two found bugs (findings 8 and 9), and the VM soft-locked during several runs.

## Checks

- [x] A fresh VM goes all the way to working stacks with one command (5 m 02 s)
- [x] Second run: `changed=0`, with no duplicate stacks created
- [x] A `-vvv` run shows no passwords or tokens: the setup token, the admin password and the Pi-hole password each appear 0 times
- [x] Stacks deployed by Ansible are fully editable in Portainer
- [x] `dig @192.168.77.50 example.com` resolves, and `doubleclick.net` returns `0.0.0.0`
- [x] `https://portainer.home.arpa`, `https://pihole.home.arpa/admin` and `https://npm.home.arpa` load with no warning
- [x] `ansible-lint`: 0 failures, 0 warnings in 43 files, `production` profile

## What broke and how I fixed it

1. **`endpointId=1` returned 404.** I'd copied the ID from examples. Mine was 3, because the browser setup had made the environment, and a fresh install makes ID 1. The role now looks the environment up by its URL.
2. **`jq: Cannot index string with string "Id"`.** Twice, Portainer had sent back an error object instead of a list: once because the login had failed and `TOKEN` held the word `null` (4 characters), once because recreating the container invalidated the JWT. Printing `${#TOKEN}` and running the call without `jq` showed the real message: `Invalid JWT token`.
3. **`Unsupported parameters … no_log`.** `no_log` was indented under the module, so Ansible passed it to `docker_container` as a module option. It belongs at task level, lined up with `register` and `when`.
4. **The setup token leaked in `-vvv` output** from "Look up the Portainer container": `docker_container_info` returns everything `docker inspect` shows, including `Config.Env`. A count of the token in `-vvv` output found 2 lines, and an `awk` script that replaced the value with `<TOKEN>` showed which task printed them. `no_log: true` fixed it, and the count went to 0.
5. **`var-naming[no-role-prefix]` on `portainer_stacks`.** The prefix ansible-lint wants is the role name plus an underscore, so a variable named exactly like the role fails. Renamed to `portainer_stacks_list`.
6. **The Pi-hole password wasn't in Vault yet.** It had only ever been typed into Portainer. Found with a `grep -c` on `ansible-vault view` output before it could break the from-zero run.
7. **A fresh install would have had no environment to deploy to.** Found by reading Portainer's start-up code before Step 3, not by a failure: see the `-H` point in Step 1.
8. **First from-zero run: only Portainer was running, and DNS was refused.** Handlers wait until the end of the play. Portainer had accepted both stacks and started deploying them in the background; then the `Restart Docker` handler (from the log-rotation config) restarted Docker and cut the deployments off. `Restart systemd-resolved`, which frees port 53, also came too late. Fix: `meta: flush_handlers` as the last task of `docker` and `dns_host`, so both restarts happen before any container starts.
9. **Second from-zero run: both stacks failed with HTTP 500.** Portainer doesn't log API errors at its normal level, and `no_log` hid the response. The cause was the role order. Portainer's container was created while `/etc/resolv.conf` still pointed at systemd-resolved's stub listener. Then `dns_host` turned the stub off, and with the flush from finding 8 that now happened before the stacks were created. Portainer's DNS pointed at a stub that no longer existed, so it couldn't resolve `github.com` to clone the repo.
   - **How it was confirmed:** the container later restarted, and Docker rebuilt its `resolv.conf` from the server's new settings (`# ExtServers: [host(192.168.77.10)]`). Re-running the playbook then created both stacks with the same request.
   - **Fix:** `dns_host` now runs before `portainer`. The third from-zero run was clean.
   - The first run had worked only by accident: the resolved restart came at the end, after the clones.
10. **I couldn't tell what restarted Portainer at 11:43.** `docker events` showed a `start` with no `die` before it. Docker keeps events in memory, so the daemon, or the whole VM, had restarted, and the first `ssh` afterwards failed with `No route to host`. Most likely the green turtle, below.

### Test-environment limitation: the green turtle, again

The VM soft-locked during several from-zero attempts. The console showed `watchdog: BUG: soft lockup - CPU#0 stuck for 22s!` (up to 42 s), and Ansible or SSH stalled until it recovered. This is the same VirtualBox-on-Hyper-V problem as Week 7. A clean run needed a few tries, so 5 m 02 s is the time of a run that didn't stall, not an average. Production runs on bare metal.

## Evidence

Step 1, before and after (no secrets):

```text
$ pcurl "$P/api/system/status" | jq
{ "Version": "2.45.3", "InstanceID": "ab60c64c-…" }
$ pcurl -o /dev/null -w '%{http_code}\n' "$P/api/users/admin/check"
204
$ echo "${#TOKEN}"
296
$ pcurl -H "$H" "$P/api/endpoints" | jq '.[] | {Id, Name, URL}'
{ "Id": 3, "Name": "local", "URL": "unix:///var/run/docker.sock" }
$ cat /tmp/resp.json      # endpointId=1
{"message":"Unable to find an environment with the specified identifier inside the database",
 "details":"Object not found inside the database (bucket=endpoints, key=1)"}
$ cat /tmp/resp.json      # second POST of the same stack
{"message":"A stack with the normalized name 'scratch-api-test' already exists", …}
$ pcurl -H "$H" "$P/api/stacks/4" | jq '{GitSourceId, AutoUpdate: .AutoUpdate.Interval}'
{ "GitSourceId": 1, "AutoUpdate": "5m" }
```

The leak, located without printing the token (finding 4):

```text
TASK [portainer_stacks : Look up the Portainer container] ****
   <192.168.77.50> (0, b'… {"changed": false, "exists": true, "cont…
TASK [portainer_stacks : Look up the Portainer container] ****
                   "PORTAINER_SETUP_TOKEN=<TOKEN>",
```

Portainer's DNS after its restart (finding 9):

```text
nameserver 127.0.0.11
# Based on host file: '/etc/resolv.conf' (internal resolver)
# ExtServers: [host(192.168.77.10)]
```

From zero (Step 3):

```text
TASK [docker : Apply pending Docker restarts now]
RUNNING HANDLER [base : Restart SSH]                         changed
RUNNING HANDLER [docker : Restart Docker]                    changed
…
TASK [dns_host : Apply pending systemd-resolved restarts now]
RUNNING HANDLER [dns_host : Restart systemd-resolved]        changed
TASK [portainer : Run the Portainer container]               changed
TASK [portainer_stacks : Create the admin user (first run only)]   changed
TASK [portainer_stacks : Create the Git source (first run only)]   changed
TASK [portainer_stacks : Create each missing stack]          changed ×2 (censored due to no_log)

test-homelab : ok=52  changed=28  unreachable=0  failed=0  skipped=1
real    5m2.177s

test-homelab : ok=46  changed=0   unreachable=0  failed=0  skipped=4

$ ssh test-homelab 'docker ps --format "{{.Names}}  {{.Status}}"'
npm  Up 2 minutes
pihole  Up 3 minutes (healthy)
unbound  Up 3 minutes
portainer  Up 3 minutes
$ dig @192.168.77.50 example.com +short
104.20.23.154
172.66.147.243
$ dig @192.168.77.50 doubleclick.net +short
0.0.0.0

vault_portainer_setup_token: 0
vault_portainer_admin_password: 0
vault_pihole_password: 0
```

## Notes for later weeks

- **Week 9:**
  - Backups: `npm_data` (proxy hosts and the uploaded certificate), Pi-hole's `/etc/pihole` and `~/.homelab-pki`. Portainer's admin, source and stacks can now be rebuilt by `site.yml`, but `portainer_data` still holds settings and history.
  - The disaster-recovery drill starts from `site.yml` and then restores the volumes.
  - Add `ansible-lint` to CI.
- **Week 9 Part D / Week 10 (production):**
  - `inventories/prod/group_vars/all/vault.yml` needs its own four secrets, and a new `vault_portainer_setup_token`.
  - `portainer_stacks_list` in prod adds `PIHOLE_DHCP_ACTIVE: "true"` to the `dns` stack.
  - `HOMELAB_IP` follows `ansible_host` (`192.168.77.10`).
  - Production's environment will be named `primary`.
- **Stretch:** update an existing stack's environment variables with a `PUT`; create NPM's proxy hosts through NPM's API to close the gap in ADR 0005.
- **Portainer 2.45.4:** Portainer's UI shows it's available. Bump `portainer_version` on its own day, then do a from-zero run.
- **Still open:** NPM 2.17.1, Docker's `live-restore`, `base_ufw_allowed` delete support, and the desktop's hosts-file line (Week 10).
