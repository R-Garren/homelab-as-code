# Role: dns_host

Prepares the host for the DNS stack in [`stacks/dns/`](../../stacks/dns/compose.yaml): Pi-hole on the host network and Unbound in a container. It frees port 53, keeps the host's own DNS working, places Unbound's config file where the stack mounts it, and opens Pi-hole's ports in UFW. The stack itself is deployed by Portainer from Git, not by this role ([ADR 0003](../../docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)). Each control is listed in [`docs/security-baseline.md`](../../docs/security-baseline.md).

## Requirements

- Ubuntu Server 26.04 LTS on the managed host (systemd-resolved, UFW).
- ansible-core 2.21 or later on the control node, with the `community.general` collection (see `requirements.yml`).
- Privilege escalation: apply the role with `become: true`.
- Apply it after the `base` role, which turns UFW on, and before the `dns` stack is deployed in Portainer, because the stack mounts the file this role creates.

## What it does

All tasks are in `tasks/main.yml`, in this order:

| Task | What it does |
| --- | --- |
| Make sure the resolved drop-in folder exists | Creates `/etc/systemd/resolved.conf.d`, mode `0755`. |
| Turn off the systemd-resolved stub listener | Writes `10-no-stub.conf` (`DNSStubListener=no`) from `templates/10-no-stub.conf.j2` and notifies `Restart systemd-resolved`. Frees `127.0.0.53:53` so Pi-hole can listen on port 53. |
| Point /etc/resolv.conf at the real upstream servers | Replaces the symlink to `stub-resolv.conf` (which lists only the stub at `127.0.0.53`) with a symlink to `/run/systemd/resolve/resolv.conf`, the servers learned from DHCP or netplan. |
| Create the Unbound config folder | `{{ dns_host_config_root }}/unbound`, mode `0755`. |
| Template the Unbound override file | Writes `pi-hole.conf` from `templates/pi-hole.conf.j2`, root-owned, mode `0644`, so the container's `unbound` user (UID 101) can read it. |
| Allow Pi-hole ports through UFW | One `allow` rule per entry in `dns_host_ufw_allowed`. |
| Allow DHCP (67/udp) only when Pi-hole DHCP is active | Adds the rule when `dns_host_dhcp_active` is true and deletes it when it's false, so switching DHCP off also closes the port. |

The handler only runs at the end of the play, after the symlink has moved, so the host never loses DNS during a run.

### The Unbound override file

`pi-hole.conf` holds the settings from [Pi-hole's Unbound guide](https://docs.pi-hole.net/guides/dns/unbound/), adapted for the `klutchell/unbound` image ([ADR 0004](../../docs/adr/0004-unbound-image.md)). The stack mounts it read-only at `/etc/unbound/custom.conf.d/pi-hole.conf`, which the image's own config includes last. It leaves out:

- `interface` and `port`: the image listens on `0.0.0.0:53` inside the container, and the stack publishes that as `127.0.0.1:5335`.
- `root-hints`: the image ships its own root hints and trust anchor.

It allows queries from `127.0.0.0/8` and `172.16.0.0/12`, the range Docker gives its networks, and sets `do-ip6: no`.

Check it against the image before deploying:

```bash
docker run --rm --entrypoint unbound-checkconf \
  -v /opt/homelab/unbound/pi-hole.conf:/etc/unbound/custom.conf.d/pi-hole.conf:ro \
  klutchell/unbound:1.26.2
```

## Variables

All are in `defaults/main.yml` and can be overridden from `group_vars` or `host_vars`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `dns_host_config_root` | `/opt/homelab` | Where stack config files live on the host. The compose file mounts them by absolute path. |
| `dns_host_web_port` | `"8081"` | Pi-hole's web UI port, opened in UFW. Must match `FTLCONF_webserver_port` in `stacks/dns/compose.yaml`. |
| `dns_host_dhcp_active` | `false` | Opens 67/udp when `true`. **Never `true` on the test VM:** it's bridged onto the home LAN, and a second DHCP server would hand out wrong addresses. Pi-hole's own DHCP setting is `PIHOLE_DHCP_ACTIVE` in Portainer; set both together. |
| `dns_host_ufw_allowed` | 53/tcp, 53/udp, `dns_host_web_port`/tcp | UFW allow rules for Pi-hole: a list of `port`, `proto` and `comment`. |

## Example

```yaml
- name: Configure homelab servers
  hosts: homelab
  become: true
  roles:
    - base
    - docker
    - portainer
    - dns_host
```

## Notes

- **The server doesn't use its own Pi-hole.** Its DNS comes from DHCP (test VM) or netplan (production), so Docker can still pull images when Pi-hole is down.
- **Removing an entry from `dns_host_ufw_allowed` doesn't close the port.** Only the DHCP rule deletes itself; any other rule needs a task with `delete: true`.
- **Unbound only reads `pi-hole.conf` when it starts.** After this role changes the file, restart the `unbound` container from Portainer.
- Pi-hole runs on the host network, so UFW applies to it. Unbound's port is published by Docker, which bypasses UFW, so it's bound to `127.0.0.1` in the compose file instead.

## License

MIT
