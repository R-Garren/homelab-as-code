# homelab-as-code

My home lab, built from scratch with Ansible and Docker: hardened, tested on a VM first, and rebuildable with one command.

**Status:** Week 6 of 12. A `base` role hardens SSH (keys only, no root login), turns on a deny-by-default UFW firewall, installs security updates automatically and sets the hostname and timezone. A `docker` role installs Docker Engine from Docker's signed repository with container log rotation. A `portainer` role runs Portainer CE 2.45.3 LTS from a pinned image. A `dns_host` role frees port 53, places Unbound's config and opens Pi-hole's firewall ports. Portainer deploys the first app stack, Pi-hole + Unbound with DNSSEC validation, straight from this repo and redeploys it when `main` changes. Secrets are encrypted with Ansible Vault.

## What this will build

One Ubuntu Server 26.04 LTS host (a ThinkPad 11e) running:

| Service | Purpose |
| --- | --- |
| Pi-hole | DNS filtering and DHCP for the home LAN |
| Unbound | Recursive, DNSSEC-validating resolver behind Pi-hole |
| Nginx Proxy Manager | HTTPS for local services, signed by a private CA |
| Portainer | Deploys the app stacks from this repo |

Ansible owns the host: packages, SSH, firewall, Docker and Portainer itself. Portainer owns the apps, deployed as Git-repository stacks from `stacks/`, so it has full control over them ([ADR 0003](docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)). Every change runs on a VirtualBox test VM before it touches production.

Pinned image versions: Portainer CE `2.45.3` (LTS), Pi-hole `2026.11.0`, Unbound `1.26.2` (`klutchell/unbound`, also pinned by digest; see [ADR 0004](docs/adr/0004-unbound-image.md)).

## Run it

Tested with ansible-core 2.21.6 (ansible 14.6.0, which bundles community.docker 5.4.0) against Ubuntu Server 26.04 LTS. Older ansible-core versions can hang on Ubuntu's sudo-rs ([ADR 0001](docs/adr/0001-keep-sudo-rs.md)).

```bash
ansible-galaxy collection install -r requirements.yml
git config core.hooksPath scripts/hooks
ansible-playbook playbooks/site.yml
```

- The sudo password comes from the encrypted `inventories/test/group_vars/all/vault.yml`, so no `-K` is needed ([ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md)).
- Ansible reads the vault password from `~/.ansible/vault_pass.txt` (set in `ansible.cfg`, mode `600`, never in the repo). Without that file, every run fails.
- `git config core.hooksPath scripts/hooks` turns on a `pre-commit` hook that refuses to commit a `vault.yml` that isn't encrypted. Run it once in every clone.

Hosts and variables live in `inventories/test/`. Each role's settings and their defaults are in [`roles/base/`](roles/base/README.md), [`roles/docker/`](roles/docker/README.md), [`roles/portainer/`](roles/portainer/README.md) and [`roles/dns_host/`](roles/dns_host/README.md). `playbooks/site.yml` is one play that applies all four roles in that order.

After the first run, open `https://<host>:9443` straight away and create the Portainer admin account. Portainer asks for a one-time setup token from its log (`docker logs portainer 2>&1 | grep -i token` on the server) and locks the setup page after a few minutes.

Run one part of the `base` role with tags: `updates`, `packages`, `system`, `ssh` or `firewall`. For example, `ansible-playbook playbooks/site.yml --tags firewall`. Preview any run with `--check --diff`.

### Deploy the DNS stack

Until Week 8 automates it, the stack is added in Portainer by hand, after `site.yml` has run (the `dns_host` role must place Unbound's config first):

1. **Sources → Add new:** this repo's URL, no authentication (it's public), polling on (`5m`).
2. **Stacks → Add stack → Repository:** name `dns`, that source, reference `refs/heads/main`, compose path `stacks/dns/compose.yaml`.
3. Add the environment variable `PIHOLE_PASSWORD` (the Pi-hole web UI password). Leave `PIHOLE_DHCP_ACTIVE` unset except in production: DHCP stays off unless it's `true`.

Pi-hole's web UI is then at `http://<host>:8081/admin`. After that, a push to `main` that changes `stacks/dns/compose.yaml` is redeployed by Portainer on its next poll.

## Security

[Security baseline](docs/security-baseline.md): every control the roles and the DNS stack apply, how to verify it on a running server, the Security+ concept it maps to, and the risks accepted along the way.

- Secrets are encrypted with Ansible Vault; the vault password file stays outside the repo, and a `pre-commit` hook refuses unencrypted vault files ([ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md)).
- Ports Docker publishes for containers bypass UFW, so containers publish on `127.0.0.1` unless a port is meant for the LAN and listed in the baseline ([details](docs/security-baseline.md#docker-bypasses-ufw)).
- Portainer mounts the Docker socket, so its admin account is effectively root on the host. Its UI on port 9443 is the only port Docker publishes to the LAN, until Nginx Proxy Manager fronts it in Week 7.
- Unbound listens on `127.0.0.1` only and validates DNSSEC; Pi-hole uses it as its only upstream. Pi-hole runs on the host network, so UFW covers its ports (DNS and the web UI), and DHCP stays off unless a variable turns it on.
- GitHub secret scanning and push protection are on.
- Private notes, the device inventory and Pi-hole backups never enter the repo (see `.gitignore`).
- Known limitation: the home network is one flat subnet until the segmentation lab in Week 11.

## Network docs

- [Current topology](docs/network/current-topology.md)
- [IP plan](docs/network/ip-plan.md)
- [Migration runbook](docs/network/migration-runbook.md)
- [Segmentation plan](docs/network/segmentation-plan.md)

## Progress

- [x] Week 0: Map the home network, re-address the LAN, remove double NAT ([docs](docs/network/))
- [x] Week 1: Foundations: control node, test VM, repo ([build log](docs/build-log/week-01.md))
- [x] Week 2: Ansible fundamentals: inventory, `ansible.cfg` and an idempotent bootstrap playbook ([build log](docs/build-log/week-02.md))
- [x] Week 3: Base host role: SSH hardening, UFW firewall, automatic security updates ([build log](docs/build-log/week-03.md), [security baseline](docs/security-baseline.md))
- [x] Week 4: Secrets with Ansible Vault + Docker Engine ([build log](docs/build-log/week-04.md), [ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md))
- [x] Week 5: Portainer, deployed by Ansible ([build log](docs/build-log/week-05.md), [ADR 0003](docs/adr/0003-ansible-owns-host-portainer-owns-apps.md))
- [x] Week 6: DNS stack: Pi-hole + Unbound deployed from Git by Portainer, DNSSEC validated ([build log](docs/build-log/week-06.md), [ADR 0004](docs/adr/0004-unbound-image.md))
- [ ] Week 7: Nginx Proxy Manager, local names and HTTPS
- [ ] Week 8: Automate Portainer through its API
- [ ] Week 9: Backups, CI and a disaster-recovery drill
- [ ] Week 10: Production cutover (v2.0.0)
- [ ] Week 11: Network segmentation lab: OPNsense VLANs + Suricata (v2.1.0)
- [ ] Week 12: WireGuard remote access (v2.2.0)

## v1 → v2

v1 ran the same services on the same ThinkPad, but much of it was set up by hand and Ansible only controlled part of the host. v2 starts over so that code owns everything, each change is tested on a VM first, and the whole server can be rebuilt with one command. v1 is archived as `homelab-iac-v1`.

## License

See [LICENSE](LICENSE).
