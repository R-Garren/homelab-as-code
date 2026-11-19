# Role: portainer

Runs [Portainer CE](https://github.com/portainer/portainer) as a Docker container from a pinned LTS image, with its data in a named volume. Ansible owns Portainer itself; Portainer owns the app stacks it deploys from Git (see [ADR 0003](../../docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)). Each control is listed in [`docs/security-baseline.md`](../../docs/security-baseline.md).

## Requirements

- Ubuntu Server 26.04 LTS on the managed host.
- ansible-core 2.21 or later on the control node, with the `community.docker` collection (see `requirements.yml`).
- Privilege escalation: apply the role with `become: true`.
- Apply it after the `docker` role, which installs Docker Engine and the Docker SDK for Python.

## What it does

All tasks are in `tasks/main.yml`, in this order:

| Task | What it does |
| --- | --- |
| Create the Portainer data volume | Creates the named volume `portainer_volume_name`. Docker keeps it under `/var/lib/docker/volumes/`, and it survives the container being removed or recreated. |
| Run the Portainer container | Pulls `portainer_image` if it's missing and runs it as `portainer_container_name`. Publishes only the HTTPS UI, mounts the Docker socket and the data volume at `/data`, and sets the restart policy. If any of these settings change, the module recreates the container; the data volume is kept. |

The role has no handlers: `community.docker.docker_container` recreates the container itself when its configuration changes.

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `portainer_version` | `"2.45.3"` | Portainer CE LTS release to run. Pinned, never `latest`. Keep it quoted. |
| `portainer_image` | `"portainer/portainer-ce:{{ portainer_version }}"` | Full image reference. |
| `portainer_container_name` | `portainer` | Container name. |
| `portainer_volume_name` | `portainer_data` | Named volume mounted at `/data`. |
| `portainer_https_port` | `9443` | Host port for the HTTPS UI. |
| `portainer_restart_policy` | `unless-stopped` | Docker restart policy. The container comes back after a crash or reboot, but stays down if you stopped it by hand. |

## Example

```yaml
- name: Configure homelab servers
  hosts: homelab
  become: true
  roles:
    - base
    - docker
    - portainer
```

## Notes

- **The Docker socket gives Portainer root-level control of the host.** Anyone who logs in to Portainer as an admin can start a privileged container and own the server. A strong admin password is the main control.
- **Port 9443 is reachable from the whole LAN.** Published ports bypass UFW (see [Docker bypasses UFW](../../docs/security-baseline.md#docker-bypasses-ufw)). This is deliberate and temporary: Week 7 puts Nginx Proxy Manager in front of Portainer over a shared Docker network.
- **Create the admin account right after the first start.** Portainer locks its setup page after a few minutes for security. If you miss it, run `docker restart portainer` and try again.
- The UI uses a self-signed certificate until Week 7, so the browser shows a warning.
- Portainer CE stays on the 2.45 LTS line: Portainer announced in September 2026 that there will be no CE build of 3.x. 2.45 still gets security and bug fixes.

## License

MIT
