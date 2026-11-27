# Role: docker

Installs Docker Engine and the Compose plugin from Docker's official apt repository, following [Docker's install guide for Ubuntu](https://docs.docker.com/engine/install/ubuntu/) step by step. It also turns on container log rotation, keeps containers running while the Docker daemon restarts (live restore), lets chosen users run Docker without `sudo`, installs what the `community.docker` modules need on the server, and creates the `proxy` network that Nginx Proxy Manager shares with the services behind it. Each control is listed in [`docs/security-baseline.md`](../../docs/security-baseline.md).

## Requirements

- Ubuntu Server 26.04 LTS on the managed host.
- ansible-core 2.21 or later on the control node.
- Privilege escalation: apply the role with `become: true`.
- Apply it after the `base` role, so the host is patched and firewalled first.

## What it does

All tasks are in `tasks/main.yml`, in this order:

| Task | What it does |
| --- | --- |
| Remove unofficial Docker packages | Removes `docker_conflicting_packages`, Ubuntu's own Docker packages that clash with Docker's. |
| Install packages needed to add Docker's repository | Installs `ca-certificates`, so the server can check Docker's HTTPS certificate, and `python3-debian`, which the repository task needs. |
| Create the apt keyrings directory | `/etc/apt/keyrings`, mode `0755`. |
| Download Docker's GPG key | Saves it to `/etc/apt/keyrings/docker.asc`, mode `0644`. |
| Add Docker's apt repository | Writes `/etc/apt/sources.list.d/docker.sources` (deb822 format), trusting only packages signed with that key. |
| Install Docker Engine | Installs `docker_packages`, refreshing the apt cache first so the new repository is seen. |
| Make sure Docker is running and starts at boot | Starts the `docker` service and enables it at boot. Docker's package normally does both, so this reports `ok` unless Docker was stopped. |
| Install the Docker SDK for Python | Installs `python3-docker` from Ubuntu, needed by the `community.docker` modules. |
| Add users to the docker group | Adds each user in `docker_users` to `docker` (`append: true`, so their other groups stay). |
| Reset the SSH connection | Makes the new group membership apply to the rest of the play. |
| Create the Docker config directory | `/etc/docker`, mode `0755`. |
| Configure the Docker daemon | Writes `/etc/docker/daemon.json` from `templates/daemon.json.j2`: log rotation and `"live-restore": true`. The file is checked with `dockerd --validate` before it's put in place, and a change notifies `Restart Docker`. |
| Create the shared proxy network | Creates the bridge network `docker_proxy_network` with a fixed subnet and gateway. NPM joins it from its stack, and the `portainer` role joins Portainer to it. |
| Apply pending Docker restarts now | `meta: flush_handlers`: runs `Restart Docker` here instead of at the end of the play. Also runs any handler an earlier role notified, such as `base`'s SSH restart. |

The `Restart Docker` handler runs only when `daemon.json` changed, at the end of this role. With live restore on, a restart leaves the running containers up. Docker still replaces the `docker-proxy` processes behind published ports, so Unbound's `127.0.0.1:5335` is closed for under a second, and NPM's ports go through the same kind of process. On the test VM (Week 10), a restart took about 1 s: 2 of about 160 lookups to Unbound were refused, and 1 of 155 through Pi-hole timed out. Without live restore, every container restarted, and 12 to 19 lookups failed over about 5 s. The exception is the restart that turns live restore on, because the daemon that stops is the one started without it: on a host that already runs containers, they stop once. A fresh install gets the setting before any container exists. Left to the end of the play, the restart came after `portainer_stacks` had asked Portainer to deploy the stacks, and cut those deployments off (Week 8). Docker's API is still down while the daemon restarts, so the role flushes its handlers before any container starts.

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `docker_conflicting_packages` | `docker.io`, `docker-compose`, `docker-compose-v2`, `docker-doc`, `docker-buildx`, `podman-docker`, `containerd`, `runc` | Packages removed before installing Docker's own. The list comes from Docker's "Uninstall old versions" step. |
| `docker_packages` | `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin` | Packages installed from Docker's repository. |
| `docker_users` | `[]` | Users added to the `docker` group. The test inventory sets `[test-homelab]`. Members of this group are effectively root. |
| `docker_log_max_size` | `"10m"` | Largest size of one container log file. Keep it quoted: Docker wants a string. |
| `docker_log_max_file` | `"3"` | Number of log files kept per container. Also a string. |
| `docker_proxy_network` | `proxy` | Name of the network shared with Nginx Proxy Manager. Must match `portainer_network` and the external network in `stacks/proxy/compose.yaml`. |
| `docker_proxy_subnet` | `"172.20.0.0/24"` | Its subnet, pinned so firewall rules can name it. Must match `dns_host_proxy_subnet`, and stay clear of the subnets in [`docs/network/ip-plan.md`](../../docs/network/ip-plan.md). |
| `docker_proxy_gateway` | `"172.20.0.1"` | The host's address on that network. NPM's `pihole.home.arpa` proxy host forwards to it, so the two must match. |

## Example

```yaml
- name: Apply docker role to homelab servers
  hosts: homelab
  become: true
  roles:
    - docker
```

## Notes

- **Published ports bypass UFW.** A port published with `-p 8080:80` is reachable from the whole LAN even though UFW denies incoming traffic. Publish on `127.0.0.1` unless a port must be reachable from the LAN. See [Docker bypasses UFW](../../docs/security-baseline.md#docker-bypasses-ufw).
- `ansible.builtin.user` creates a user that doesn't exist, so a typo in `docker_users` creates a new account instead of failing. Watch for `changed` on that task.
- **Changing the proxy network's subnet or gateway later means recreating it.** Docker can't change a network's addresses in place, and it refuses to remove a network that containers are still attached to. Stop NPM and Portainer first.
- Log settings apply only to containers created after Docker restarts. Existing containers keep their old settings until they're recreated.
- **Live restore covers patch upgrades only.** Docker supports it from one patch release to the next (`29.10.1` to `29.10.2`), not to a new major release (`29.x` to `30.x`), where the new daemon may not reconnect to the running containers and they have to be stopped by hand. Docker's packages aren't in the unattended-upgrades origins (only Ubuntu's security updates are), so every Docker upgrade is a manual `apt` step: plan a major upgrade like a container restart.
- The handler restarts Docker rather than reloading it. `systemctl reload docker` would turn live restore on without a restart, but a reload doesn't apply every setting in `daemon.json` (the log settings among them), and with live restore on, a restart no longer stops the containers.
- While the daemon is down, containers write their logs to a pipe that it normally reads. If it stayed down for a long time, a busy container could fill the 64 KB pipe and block until the daemon is back. A restart takes seconds.
- Live restore doesn't work in Swarm mode, which this lab doesn't use.

## License

MIT
