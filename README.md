# homelab-as-code

My home lab, built from scratch with Ansible and Docker: hardened, tested on a VM first, and rebuildable with one command.

**Status:** Week 1 of 12. The control node and test VM are ready; nothing is deployed by this repo yet.

## What this will build

One Ubuntu Server 26.04 LTS host (a ThinkPad 11e) running:

| Service | Purpose |
| --- | --- |
| Pi-hole | DNS filtering and DHCP for the home LAN |
| Unbound | Recursive, DNSSEC-validating resolver behind Pi-hole |
| Nginx Proxy Manager | HTTPS for local services, signed by a private CA |
| Portainer | Deploys the app stacks from this repo |

Ansible owns the host: packages, SSH, firewall and Docker. Portainer owns the apps, deployed as Git-repository stacks from `stacks/`. Every change runs on a VirtualBox test VM before it touches production. Pinned image versions will be listed here once the stacks land.

## Network docs

- [Current topology](docs/network/current-topology.md)
- [IP plan](docs/network/ip-plan.md)
- [Migration runbook](docs/network/migration-runbook.md)
- [Segmentation plan](docs/network/segmentation-plan.md)

## Progress

- [x] Week 0: Map the home network, re-address the LAN, remove double NAT ([docs](docs/network/))
- [x] Week 1: Foundations: control node, test VM, repo ([build log](docs/build-log/week-01.md))
- [ ] Week 2: Ansible fundamentals
- [ ] Week 3: Base host role (hardening)
- [ ] Week 4: Secrets with Ansible Vault + Docker Engine
- [ ] Week 5: Portainer, deployed by Ansible
- [ ] Week 6: DNS stack (Pi-hole + Unbound from Git)
- [ ] Week 7: Nginx Proxy Manager, local names and HTTPS
- [ ] Week 8: Automate Portainer through its API
- [ ] Week 9: Backups, CI and a disaster-recovery drill
- [ ] Week 10: Production cutover (v2.0.0)
- [ ] Week 11: Network segmentation lab: OPNsense VLANs + Suricata (v2.1.0)
- [ ] Week 12: WireGuard remote access (v2.2.0)

## Security notes

- Secrets will live in Ansible Vault; the vault password file stays outside the repo.
- GitHub secret scanning and push protection are on.
- Private notes, the device inventory and Pi-hole backups never enter the repo (see `.gitignore`).
- Known limitation: the home network is one flat subnet until the segmentation lab in Week 11.

## v1 → v2

v1 ran the same services on the same ThinkPad, but much of it was set up by hand and Ansible only controlled part of the host. v2 starts over so that code owns everything, each change is tested on a VM first, and the whole server can be rebuilt with one command. v1 is archived as `homelab-iac-v1`.

## License

See [LICENSE](LICENSE).
