# homelab-as-code

My home lab, built from scratch with Ansible and Docker: hardened, tested on a VM first, and rebuildable with one command.

**Status:** Week 4 of 12. A `base` role hardens SSH (keys only, no root login), turns on a deny-by-default UFW firewall, installs security updates automatically and sets the hostname and timezone. A `docker` role installs Docker Engine from Docker's signed repository with container log rotation. Secrets are encrypted with Ansible Vault. No services are deployed yet.

## What this will build

One Ubuntu Server 26.04 LTS host (a ThinkPad 11e) running:

| Service | Purpose |
| --- | --- |
| Pi-hole | DNS filtering and DHCP for the home LAN |
| Unbound | Recursive, DNSSEC-validating resolver behind Pi-hole |
| Nginx Proxy Manager | HTTPS for local services, signed by a private CA |
| Portainer | Deploys the app stacks from this repo |

Ansible owns the host: packages, SSH, firewall and Docker. Portainer owns the apps, deployed as Git-repository stacks from `stacks/`. Every change runs on a VirtualBox test VM before it touches production. Pinned image versions will be listed here once the stacks land.

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

Hosts and variables live in `inventories/test/`. Each role's settings and their defaults are in [`roles/base/`](roles/base/README.md) and [`roles/docker/`](roles/docker/README.md).

Run one part of the `base` role with tags: `updates`, `packages`, `system`, `ssh` or `firewall`. For example, `ansible-playbook playbooks/site.yml --tags firewall`. Preview any run with `--check --diff`.

## Security

[Security baseline](docs/security-baseline.md): every control the `base` and `docker` roles apply, how to verify it on a running server, the Security+ concept it maps to, and the risks accepted along the way.

- Secrets are encrypted with Ansible Vault; the vault password file stays outside the repo, and a `pre-commit` hook refuses unencrypted vault files ([ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md)).
- Ports Docker publishes for containers bypass UFW, so containers publish on `127.0.0.1` unless a port is meant for the LAN and listed in the baseline ([details](docs/security-baseline.md#docker-bypasses-ufw)).
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
- [ ] Week 5: Portainer, deployed by Ansible
- [ ] Week 6: DNS stack (Pi-hole + Unbound from Git)
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
