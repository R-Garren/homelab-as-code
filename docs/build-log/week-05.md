# Week 5: Portainer, deployed by Ansible

**Date:** 18 Nov 2026 · **Status:** done

## Goal

Deploy Portainer CE with Ansible from a pinned image, then reproduce v1's "control limited" warning on purpose, to understand why it happens and record how v2 avoids it (ADR 0003).

## Decided before starting

- **Tool: stay on Portainer CE for v2.0.** I looked at Komodo, Arcane and Dockge. Komodo is the strongest infrastructure-as-code option (declarative TOML synced from Git), but it's more complex and also advises absolute bind-mount paths. It's a possible project after v2.0.
- **Port 9443 is published to the LAN** and listed in the security baseline as LAN-facing, temporary until Week 7, when Nginx Proxy Manager fronts Portainer over a shared Docker network. No `DOCKER-USER` rule and no DHCP reservation for the desktop yet; the desktop gets a reservation when a source-limiting rule needs one.
- **`site.yml` became one play,** `Configure homelab servers`, with `roles: [base, docker]`, following a note from Week 4. Facts are now gathered once instead of once per play, so the run reported `ok=23 changed=0`, one `ok` fewer than before. `ansible-lint` stayed clean at the `production` profile (24 files).
- **Snapshot:** `05-before-portainer` holds the Week 4 end state.

## What I did

### Step 1: The `portainer` role

#### Layout

`roles/portainer/` has `defaults/`, `meta/`, `tasks/` and a README. I made a `handlers/` folder at first and then removed it: `community.docker.docker_container` recreates the container itself whenever its settings change, so there is nothing for a handler to restart. `templates/` wasn't needed either: Portainer has no config file, it keeps everything in its database.

#### Defaults

| Variable | Value | Why |
| --- | --- | --- |
| `portainer_version` | `"2.45.3"` | The current LTS release on [Portainer's GitHub releases](https://github.com/portainer/portainer/releases). Quoted, so a version like `2.50` can't be read as a number and lose its trailing zero. |
| `portainer_image` | `"portainer/portainer-ce:{{ portainer_version }}"` | The full image reference, built from the version. |
| `portainer_container_name` | `portainer` | Weeks 7 and 8 refer to it. |
| `portainer_volume_name` | `portainer_data` | Named volume mounted at `/data`. |
| `portainer_https_port` | `9443` | Host port for the HTTPS UI. |
| `portainer_restart_policy` | `unless-stopped` | See below. |

**Version research.** The `release/2.45` branch already had a commit bumping it to 2.45.4, but 2.45.3 was still the latest published release, so that's what is pinned. While checking, I found that Portainer announced in September 2026 that there will be no CE build of 3.x. CE stays on the 2.45 LTS line and still gets security and bug fixes. That doesn't change this week's plan, because 2.45 LTS is what I'd pin anyway, but it's recorded in ADR 0003 and the security baseline.

**Restart policy.** `always` brings a container back whenever Docker starts, even if I had stopped it on purpose with `docker stop`. `unless-stopped` also restarts it after a crash or a reboot, but respects a manual stop. For an admin UI I might want to switch off for a while, `unless-stopped` fits.

#### Tasks

`tasks/main.yml` has two tasks:

1. **Create the Portainer data volume:** `community.docker.docker_volume` with `name: "{{ portainer_volume_name }}"` and `state: present`.
2. **Run the Portainer container:** `community.docker.docker_container` with the name, image and restart policy from variables, `state: started`, one published port `"{{ portainer_https_port }}:9443"`, and two volumes: `/var/run/docker.sock:/var/run/docker.sock` and `"{{ portainer_volume_name }}:/data"`.

Only the host side of the port is a variable. Inside the container Portainer always listens on 9443. Port 8000, used by Edge agents, isn't published.

`state: started` means "exists and is running". `state: present` would create the container without starting it. The module pulls the image when it's missing, so no separate pull task is needed.

`portainer` was added to `site.yml` after `docker`.

### Step 2: First login

The first run created the volume and the container. I opened `https://192.168.77.50:9443` and accepted the self-signed certificate warning.

**New in 2.45: a setup token.** The first-run page asked for a **Setup token** as well as the admin username and password. Portainer prints a one-time token to its own log on first start, so only someone with access to the server can claim the admin account:

```bash
ssh test-homelab 'docker logs portainer 2>&1 | grep -i token'
```

`2>&1` includes error output in the search, and the `test-homelab` user doesn't need `sudo` for `docker` because it's in the `docker` group. The password has to be at least 12 characters; it's stored in Bitwarden until Week 8 moves it into Vault.

On the next screen, **Set up Edge Compute**, I left the toggle off and clicked **Skip**. Edge Compute manages remote hosts through Edge agents, which is what port 8000 is for, and there is only one local host.

The tour found the `local` environment, the `portainer` container and the `portainer_data` volume, which Docker keeps at `/var/lib/docker/volumes/portainer_data/_data`. The footer shows `Portainer Community Edition 2.45.3 LTS`.

The second run reported `ok=25 changed=0`. The one `skipped` is the docker role's "Reset the SSH connection" task, which only runs when the group membership changes.

**Reboot test.** After `ssh -t test-homelab 'sudo reboot'`, Portainer came back on its own and showed the login page, not the setup page. That proves two things: the restart policy brought the container back, and the admin account survived, because it lives in the volume.

### Step 3: Reproduce v1's problem on purpose

I deployed the same one-container `traefik/whoami` stack twice, both bound to `127.0.0.1` (the Week 4 port rule):

1. **From the shell** on the VM: a `compose.yaml` in `~/whoami-cli`, published on `127.0.0.1:8081:80`, started with `docker compose up -d`.
2. **From Portainer:** Stacks → Add stack → Web editor, named `whoami-portainer`, published on `127.0.0.1:8082:80`.

| Stack | Created with | What Portainer showed | Page URL |
| --- | --- | --- | --- |
| `whoami-cli` | `docker compose up -d` | "This stack was created outside of Portainer. Control over this stack is limited." No Editor tab. | `…/stacks/whoami-cli?type=2&external=true` |
| `whoami-portainer` | Web editor | Stack and Editor tabs, Stop this stack, Delete this stack, Create template from stack | `…/stacks/whoami-portainer?id=1&type=2&regular=true&…` |

That's exactly v1's screenshot. Screenshots: [limited control](../img/week-05-stack-limited-control.png), [full control](../img/week-05-stack-full-control.png).

**Root cause:** Portainer fully manages only stacks it has a record and a compose file for in its own database, kept under `/data/compose/<stack id>` in its volume (stack `1` here). A stack started anywhere else is only recognised from the `com.docker.compose.project` label that Compose puts on its containers, so Portainer has no compose file to edit or redeploy and can only start and stop the containers.

Cleanup: **Delete this stack** in Portainer for `whoami-portainer`. On the VM, `docker compose down` in `~/whoami-cli` (it has to run in the folder that holds `compose.yaml`), then `rm -r ~/whoami-cli` and `docker image rm traefik/whoami`.

### Step 4: ADR 0003

[`docs/adr/0003-ansible-owns-host-portainer-owns-apps.md`](../adr/0003-ansible-owns-host-portainer-owns-apps.md) compares three ways to deploy the apps: Ansible running `docker compose` (limited control, v1's problem), Portainer's web editor (full control but not in Git), and Portainer Git-repository stacks (full control and in Git). The decision is Git-repository stacks: Ansible owns the host and Portainer itself, Portainer owns the app stacks.

**Research item: relative bind-mount paths.** Portainer CE doesn't support them for Git stacks. The **Enable relative path volumes** option is Business Edition only. CE clones the repo inside its own container under `/data/compose/<stack id>`, so a path like `./config` points at a folder that doesn't exist on the host. Business Edition wouldn't help either: users report that stacks deployed with relative paths show as "Limited control", because a helper container creates them (GitHub discussion #9795). So Ansible places config files under `/opt/homelab/<stack>/`, and compose files mount them by full path.

### Security baseline

[`docs/security-baseline.md`](../security-baseline.md) gained:

- **Three controls:** a pinned, supported Portainer release; only port 9443 published, with Edge Compute off; and the setup token, which stops the admin account being claimed remotely.
- **The first entry in "Ports published to the LAN":** `9443/tcp`, with no source filter, until Week 7.
- **Four accepted risks:** the Docker socket mount (a Portainer admin is effectively root on the host), 9443 open to the whole LAN, the self-signed certificate, and CE being frozen on 2.45 LTS.
- **A known gap and a plan:** the Portainer admin password is set by hand and kept only in Bitwarden until Week 8.

## Checks

- [x] Portainer is reachable at `https://192.168.77.50:9443`
- [x] On the VM, `ss -tulpn` shows nothing listening on 8000
- [x] Reboot the VM: Portainer comes back on its own (restart policy), and the admin account survives (volume)
- [x] The second run shows `changed=0`, and the image tag is pinned (`2.45.3`)
- [x] ADR 0003 and both screenshots are in the repo
- [x] `ansible-lint`: 0 failures, 0 warnings, `production` profile

## What broke and how I fixed it

1. **The first `defaults/main.yml` didn't match the plan.** It had `portainer_version: latest`, which the guide rules out, plus `portainer_data_dir: /var/lib/portainer` and `portainer_config_dir: /etc/portainer`. Portainer uses a named volume mounted at `/data` and has no separate config directory. Caught in review before the first run.
2. **`meta/main.yml` was copied from the `docker` role** and still described Docker Engine from Docker's apt repository. Rewritten to describe this role.
3. **`portainer_host_port` is undefined.** The task used a variable name that didn't exist; the defaults call it `portainer_https_port`. Ansible would have stopped at the container task. Caught in review, the same kind of mistake as Week 4's `docker_packeges`.
4. **The task name didn't match the README.** `Run Portainer container` in the task, "Run the Portainer container" in the README table. Renamed the task so the run output and the docs match.
5. **The setup token.** The guide expected only a setup timeout. Portainer 2.45 also asks for a token from its log (Step 2).
6. **`sudo: A terminal is required to authenticate`** again, on `ssh test-homelab 'sudo ss -tulpn …'`. Same fix as Week 4: `ssh -t` gives the command a terminal, so `sudo` can ask for the password.
7. **The test `compose.yaml` had two mistakes:** `traefil/whoami` (that image doesn't exist, so the pull would fail) and container port `443`. `whoami` serves plain HTTP on port 80, so `curl` would have reached nothing. Fixed before running.

## Evidence

Second run:

```text
TASK [portainer : Create the Portainer data volume]
ok: [test-homelab]
TASK [portainer : Run Portainer container]
ok: [test-homelab]

test-homelab : ok=25  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

(This run was before the task was renamed to `Run the Portainer container`.)

Volume, as Portainer shows it:

```text
portainer_data   local   /var/lib/docker/volumes/portainer_data/_data
```

Stack URLs:

```text
whoami-cli        /#!/3/docker/stacks/whoami-cli?type=2&external=true
whoami-portainer  /#!/3/docker/stacks/whoami-portainer?id=1&type=2&regular=true&orphaned=false&orphanedRunning=false
```
Listening ports after the reboot:

```text
$ ssh -t test-homelab 'sudo ss -tulpn | grep -E ":(8000|9443)"'
tcp   LISTEN 0      4096      0.0.0.0:9443      0.0.0.0:*    users:(("docker-proxy",pid=1562,fd=8))
```

- `ansible-lint`: 0 failures, 0 warnings in 28 files, `production` profile.

## Notes for later weeks

- **Week 6:** the first real Git stack. Compose files go in `stacks/<name>/compose.yaml`; config files go under `/opt/homelab/<stack>/`, placed by Ansible before the stack is deployed, and are mounted by absolute path.
- **Week 7:** put Portainer on a Docker network shared with Nginx Proxy Manager, stop publishing 9443, and remove it from "Ports published to the LAN".
- **Week 8:** creating the admin through `POST /api/users/admin/init` will probably need the setup token too. Check what the API expects in 2.45.
- **Upgrades:** 2.45.4 is coming (already bumped on the `release/2.45` branch). An upgrade means changing `portainer_version` and running `site.yml`; the module recreates the container and the volume keeps the data.
- **The CE freeze:** watch for the end of 2.45 LTS support. The compose files in `stacks/` are plain Compose, so they can move to another manager such as Komodo.
- **Still open from Week 4:** Docker's `live-restore` setting, before production.
