# Role: portainer

Runs [Portainer CE](https://github.com/portainer/portainer) as a Docker container from a pinned LTS image, with its data in a named volume. Ansible owns Portainer itself; Portainer owns the app stacks it deploys from Git (see [ADR 0003](../../docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)). The [`portainer_stacks`](../portainer_stacks/README.md) role then creates the admin account and the stacks through Portainer's API. Each control is listed in [`docs/security-baseline.md`](../../docs/security-baseline.md).

## Requirements

- Ubuntu Server 26.04 LTS on the managed host.
- ansible-core 2.21 or later on the control node, with the `community.docker` collection (see `requirements.yml`).
- Privilege escalation: apply the role with `become: true`.
- Apply it after the `docker` role, which installs Docker Engine and the Docker SDK for Python, and after `dns_host`. Docker gives a container its DNS servers when it's created, so a Portainer created before `dns_host` turns off systemd-resolved's stub listener keeps forwarding DNS to a stub that's gone, and can't clone stacks from GitHub.

## What it does

All tasks are in `tasks/main.yml`, in this order:

| Task | What it does |
| --- | --- |
| Create the Portainer data volume | Creates the named volume `portainer_volume_name`. Docker keeps it under `/var/lib/docker/volumes/`, and it survives the container being removed or recreated. |
| Run the Portainer container | Pulls `portainer_image` if it's missing and runs it as `portainer_container_name` on the `portainer_network` network. Publishes **no** ports, mounts the Docker socket and the data volume at `/data`, and sets the restart policy. Starts Portainer with `-H portainer_docker_endpoint` and sets `PORTAINER_SETUP_TOKEN` to `portainer_setup_token`. If any of these settings change, the module recreates the container; the data volume is kept. `no_log`, because the module's output includes the environment. |

The role has no handlers: `community.docker.docker_container` recreates the container itself when its configuration changes.

**No published ports.** Nginx Proxy Manager reaches Portainer as `portainer:9443` over the shared network, and the browser uses `https://portainer.home.arpa`. The task sets `published_ports: []` together with `comparisons: { published_ports: strict }`. The empty list alone isn't enough: by default the module only checks that the ports you list are published, and an empty list always passes, so a container that still published 9443 was reported `ok`.

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `portainer_version` | `"2.45.3"` | Portainer CE LTS release to run. Pinned, never `latest`. Keep it quoted. |
| `portainer_image` | `"portainer/portainer-ce:{{ portainer_version }}"` | Full image reference. |
| `portainer_container_name` | `portainer` | Container name. |
| `portainer_volume_name` | `portainer_data` | Named volume mounted at `/data`. |
| `portainer_restart_policy` | `unless-stopped` | Docker restart policy. The container comes back after a crash or reboot, but stays down if you stopped it by hand. |
| `portainer_network` | `proxy` | Docker network shared with Nginx Proxy Manager. Created by the `docker` role, so it must match `docker_proxy_network`. |
| `portainer_setup_token` | `""` | Custom one-time setup token for creating the first admin ([ADR 0006](../../docs/adr/0006-portainer-admin-setup-token.md)). Set it from Vault (`vault_portainer_setup_token`); `portainer_stacks_setup_token` must be the same value. Empty means Portainer makes a random one and prints it in its log. Ignored once an admin exists. |
| `portainer_docker_endpoint` | `"unix:///var/run/docker.sock"` | Docker address passed with `-H`. Portainer creates its environment (named `primary`) from it right after the admin is created; without `-H`, 2.45 creates none. Must match `portainer_stacks_docker_url`. |

## Example

```yaml
- name: Configure homelab servers
  hosts: homelab
  become: true
  roles:
    - base
    - docker
    - dns_host
    - portainer
    - portainer_stacks
```

## Notes

- **The Docker socket gives Portainer root-level control of the host.** Anyone who logs in to Portainer as an admin can start a privileged container and own the server. A strong admin password is the main control.
- **Nothing reaches Portainer directly from the LAN.** It's reached only through Nginx Proxy Manager, with a certificate from the private CA ([ADR 0005](../../docs/adr/0005-private-ca.md)). Behind NPM, Portainer still uses its own self-signed certificate; NPM doesn't check it, and the hop never leaves the server.
- **The admin account is created by `portainer_stacks`,** seconds after the first start, with the setup token from Vault. The token stays in the container's environment (`docker inspect` shows it), but Portainer ignores it once an admin exists.
- **Manual fallback, if `portainer_stacks` isn't used:** on a fresh server there's no published port and NPM isn't set up yet. On the server, `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' portainer` prints Portainer's address on the `proxy` network (for example `172.20.0.2`). From the control node, `ssh -N -L 9443:172.20.0.2:9443 <host>` forwards it, and `https://localhost:9443` opens the setup page. The container name doesn't work in the tunnel, because only Docker's own DNS knows it. The page asks for the setup token: `portainer_setup_token` if it's set, otherwise the one in Portainer's log. Portainer locks the setup page after 5 minutes; if you miss it, run `docker restart portainer` and try again.
- Portainer CE stays on the 2.45 LTS line: Portainer announced in September 2026 that there will be no CE build of 3.x. 2.45 still gets security and bug fixes.

## License

MIT
