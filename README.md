# homelab-as-code

My home lab, built from scratch with Ansible and Docker: hardened, tested on a VM first, and rebuildable with one command.

**Status:** Week 8 of 12. A `base` role hardens SSH (keys only, no root login), turns on a deny-by-default UFW firewall, installs security updates automatically and sets the hostname and timezone. A `docker` role installs Docker Engine from Docker's signed repository with container log rotation, and creates the network the reverse proxy shares with the services behind it. A `portainer` role runs Portainer CE 2.45.3 LTS from a pinned image, with no published ports. A `dns_host` role frees port 53, places Unbound's config and opens Pi-hole's firewall ports. A `portainer_stacks` role creates Portainer's admin account, its Git source and the stacks through Portainer's API, with the passwords from Vault. Portainer deploys two app stacks straight from this repo and redeploys them when `main` changes: Pi-hole + Unbound with DNSSEC validation, and Nginx Proxy Manager. The admin UIs are reached only through NPM, at `https://<name>.home.arpa`, with certificates from a private CA built by `playbooks/pki.yml`. Secrets are encrypted with Ansible Vault.

**Bare Ubuntu to working DNS in about 2 minutes, with one command:** from a fresh Ubuntu Server install, `ansible-playbook playbooks/site.yml` took 2 m 05 s on the test VM to bring up Pi-hole, Unbound, Nginx Proxy Manager and Portainer, with no clicks in Portainer. The only manual step left is NPM's three proxy hosts, about 3 minutes.

## What this will build

One Ubuntu Server 26.04 LTS host (a ThinkPad 11e) running:

| Service | Purpose |
| --- | --- |
| Pi-hole | DNS filtering and DHCP for the home LAN |
| Unbound | Recursive, DNSSEC-validating resolver behind Pi-hole |
| Nginx Proxy Manager | HTTPS for local services, signed by a private CA |
| Portainer | Deploys the app stacks from this repo |

Ansible owns the host: packages, SSH, firewall, Docker and Portainer itself. Portainer owns the apps, deployed as Git-repository stacks from `stacks/`, so it has full control over them ([ADR 0003](docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)). Every change runs on a Hyper-V test VM before it touches production ([ADR 0007](docs/adr/0007-test-vm-on-hyper-v.md)).

Pinned image versions: Portainer CE `2.45.3` (LTS), Pi-hole `2026.11.0`, Unbound `1.26.2` (`klutchell/unbound`, also pinned by digest; see [ADR 0004](docs/adr/0004-unbound-image.md)), Nginx Proxy Manager `2.17.0` (also pinned by digest).

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

Hosts and variables live in `inventories/test/`. Each role's settings and their defaults are in [`roles/base/`](roles/base/README.md), [`roles/docker/`](roles/docker/README.md), [`roles/dns_host/`](roles/dns_host/README.md), [`roles/portainer/`](roles/portainer/README.md) and [`roles/portainer_stacks/`](roles/portainer_stacks/README.md). `playbooks/site.yml` is one play that applies all five roles in that order. The order matters: `dns_host` must run before `portainer` (see its README).

The vault file needs four secrets: `vault_ansible_become_password`, `vault_portainer_setup_token` (a long random value, for example from `openssl rand -hex 32`), `vault_portainer_admin_password` (at least 12 characters) and `vault_pihole_password`.

Run one part of the `base` role with tags: `updates`, `packages`, `system`, `ssh` or `firewall`. For example, `ansible-playbook playbooks/site.yml --tags firewall`. Preview any run with `--check --diff`.

### What Portainer gets from Ansible

The `portainer_stacks` role ([ADR 0006](docs/adr/0006-portainer-admin-setup-token.md)) creates, only where they're missing:

1. **The admin account** (`admin`), with the setup token and password from Vault, seconds after Portainer's first start.
2. **The Git source** for this repo, polling `main` every 5 minutes.
3. **The stacks** in `portainer_stacks_list`: `dns` (`stacks/dns/compose.yaml`, with `PIHOLE_PASSWORD` from Vault and `HOMELAB_IP` from the inventory's `ansible_host`) and `proxy` (`stacks/proxy/compose.yaml`). `PIHOLE_DHCP_ACTIVE` is left unset except in production: DHCP stays off unless it's `true`.

After that, a push to `main` that changes a compose file is redeployed by Portainer on its next poll. Pi-hole answers `npm.home.arpa`, `pihole.home.arpa` and `portainer.home.arpa` with `HOMELAB_IP`. The role never updates a stack that already exists, so changing a stack's environment variables is still done in Portainer.

### Set up HTTPS (private CA + Nginx Proxy Manager)

1. **Build the CA and the server certificate** on the control node: `ansible-playbook playbooks/pki.yml`. Keys go to `~/.homelab-pki/`, outside the repo; the CA key's passphrase comes from the encrypted `playbooks/vars/pki/vault.yml` ([ADR 0005](docs/adr/0005-private-ca.md)). Import `ca.crt` into each device that should trust the sites.
2. **The proxy stack** is created by `site.yml` (above).
3. **Open NPM's admin UI** through a tunnel (`ssh -N -L 8181:127.0.0.1:81 <host>`, then `http://localhost:8181`) and create the admin account.
4. **Upload the certificate** (Certificates → Custom: `server.key` and `server.crt`) and add the proxy hosts below, each with that certificate, Force SSL and HTTP/2 on:

   | Name | Forward to |
   | --- | --- |
   | `npm.home.arpa` | `http://127.0.0.1:81` |
   | `pihole.home.arpa` | `http://172.20.0.1:8081` (`docker_proxy_gateway`) |
   | `portainer.home.arpa` | `https://portainer:9443`, websockets on |

NPM's proxy hosts live in its own database, not in this repo; that gap is explained in ADR 0005.

## Security

[Security baseline](docs/security-baseline.md): every control the roles and the DNS stack apply, how to verify it on a running server, the Security+ concept it maps to, and the risks accepted along the way.

- Secrets are encrypted with Ansible Vault; the vault password file stays outside the repo, and a `pre-commit` hook refuses unencrypted vault files ([ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md)).
- Ports Docker publishes for containers bypass UFW, so containers publish on `127.0.0.1` unless a port is meant for the LAN and listed in the baseline ([details](docs/security-baseline.md#docker-bypasses-ufw)).
- Ansible creates Portainer's admin account through the API with a setup token from Vault, so a fresh instance can't be claimed by anyone else. Tasks that carry secrets use `no_log`, checked with a `-vvv` run that must print no secret ([ADR 0006](docs/adr/0006-portainer-admin-setup-token.md)).
- Portainer mounts the Docker socket, so its admin account is effectively root on the host. It publishes no ports; it's reached only through Nginx Proxy Manager, whose 80 and 443 are the only ports Docker publishes to the LAN.
- HTTPS for every admin UI uses certificates from a private CA. Its key is passphrase-encrypted, kept outside Git, and can only sign names under `home.arpa` ([ADR 0005](docs/adr/0005-private-ca.md)).
- Unbound listens on `127.0.0.1` only and validates DNSSEC; Pi-hole uses it as its only upstream. Pi-hole runs on the host network, so UFW covers its ports: DNS for the LAN, and the web UI only from the proxy network. DHCP stays off unless a variable turns it on.
- GitHub secret scanning and push protection are on.
- Private notes, the device inventory and Pi-hole backups never enter the repo (see `.gitignore`).
- Known limitation: the home network is one flat subnet. VLAN segmentation is out of scope for v2 ([segmentation plan](docs/network/segmentation-plan.md)).

## Network docs

- [Current topology](docs/network/current-topology.md)
- [IP plan](docs/network/ip-plan.md)
- [Migration runbook](docs/network/migration-runbook.md)
- [Segmentation plan](docs/network/segmentation-plan.md) (future project, not part of v2)

## Progress

- [x] Week 0: Map the home network, re-address the LAN, remove double NAT ([docs](docs/network/))
- [x] Week 1: Foundations: control node, test VM, repo ([build log](docs/build-log/week-01.md))
- [x] Week 2: Ansible fundamentals: inventory, `ansible.cfg` and an idempotent bootstrap playbook ([build log](docs/build-log/week-02.md))
- [x] Week 3: Base host role: SSH hardening, UFW firewall, automatic security updates ([build log](docs/build-log/week-03.md), [security baseline](docs/security-baseline.md))
- [x] Week 4: Secrets with Ansible Vault + Docker Engine ([build log](docs/build-log/week-04.md), [ADR 0002](docs/adr/0002-secrets-with-ansible-vault.md))
- [x] Week 5: Portainer, deployed by Ansible ([build log](docs/build-log/week-05.md), [ADR 0003](docs/adr/0003-ansible-owns-host-portainer-owns-apps.md))
- [x] Week 6: DNS stack: Pi-hole + Unbound deployed from Git by Portainer, DNSSEC validated ([build log](docs/build-log/week-06.md), [ADR 0004](docs/adr/0004-unbound-image.md))
- [x] Week 7: Nginx Proxy Manager, local names and HTTPS from a private CA ([build log](docs/build-log/week-07.md), [ADR 0005](docs/adr/0005-private-ca.md))
- [x] Week 8: Portainer automated through its API: admin, Git source and stacks with no clicks, bare Ubuntu to working DNS in about 5 minutes ([build log](docs/build-log/week-08.md), [ADR 0006](docs/adr/0006-portainer-admin-setup-token.md))
- [ ] Week 9: Backups, CI and a disaster-recovery drill
- [ ] Week 10: Production cutover (v2.0.0)
- [ ] Week 11: WireGuard remote access (v2.1.0)

## v1 → v2

v1 ran the same services on the same ThinkPad, but much of it was set up by hand and Ansible only controlled part of the host. v2 starts over so that code owns everything, each change is tested on a VM first, and the whole server can be rebuilt with one command. v1 is archived as `homelab-iac-v1`.

## License

See [LICENSE](LICENSE).
