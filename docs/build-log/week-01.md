# Week 1: Foundations

**Date:** 17 Nov 2026 · **Status:** done

## Goal

A clean Ansible control node in WSL, a fresh Ubuntu Server 26.04 test VM I can reach with an SSH key, and this repo ready for Ansible work.

## What I did

### Retired v1

- Renamed the old repo to `homelab-iac-v1` and archived it (kept private).

### Control node (WSL, Ubuntu 24.04)

- Removed the apt copy of Ansible so pipx is the only one: ansible 14.6.0 (ansible-core 2.21.6) and ansible-lint 26.11.0, both in `~/.local/bin`.
- One SSH key per job, each with a passphrase: `github_wsl2` for GitHub, `homelab_v2` for v2 servers, and `ansible_homelab` for v1 only (retired in Week 10). There is no generic `id_ed25519`. `keychain` loads the keys once per login.
- `~/.ssh/config` has one `Host` entry per target with `IdentitiesOnly yes`, so each host is offered only the key meant for it.
- Git commits use my GitHub noreply address, and GitHub blocks any push that would expose my real email.
- VS Code with the WSL and Red Hat Ansible extensions. I skipped the extension's offer to install its own Ansible package, which would have made a third copy.
- Cloned this repo to `~/code/homelab-as-code` over SSH.

### Test VM: `test-homelab`

| Setting | Value |
| --- | --- |
| OS | Ubuntu Server 26.04.1 LTS (Resolute Raccoon) |
| CPU / RAM / disk | 2 vCPUs / 4096 MB / 25 GB dynamic |
| Network | Bridged adapter; `192.168.77.50` from a Pi-hole DHCP reservation |
| Access | key login with `homelab_v2` |
| Baseline | snapshot `01-fresh-install` |

- Installed with OpenSSH server and without the Docker snap; Ansible installs Docker properly in Week 4.
- Copied my key with `ssh-copy-id`, then checked that the host key fingerprint on the VM console matched the one WSL saved.

### Linux tour: what each command told me

- `cat /etc/os-release`: Ubuntu 26.04.1 LTS, codename `resolute`.
- `ip -br a` and `ip route`: interface `enp0s3`, `192.168.77.50/24` from DHCP, default gateway `192.168.77.1`. It also has an IPv6 ULA address, so some device on the LAN sends router advertisements even though IPv6 is off on the gateway.
- `ss -tulpn`: only SSH (port 22) listens on the network. `systemd-resolved` holds `127.0.0.53:53`, which Pi-hole will need in Week 6.
- `systemctl status ssh`: running, but listed as `disabled`, because `ssh.socket` starts it on the first connection (socket activation).
- `journalctl -u ssh -n 30`: shows the one password login from `ssh-copy-id`, then key logins from my desktop.
- `ls -la ~/.ssh`: `authorized_keys` is 600 and `~/.ssh` is 700. sshd refuses keys from files other users can write to.
- `sudo -l`: my user can run anything, with a password. The `[sudo: authenticate]` prompt shows this is sudo-rs, not classic sudo.
- `df -h` and `free -h`: 3.3 GiB RAM and 2 GiB swap; the root disk was only 12G (see below).

### Repo

- README with the plan and a progress checklist, this build log, and an expanded `.gitignore`.
- Renamed `README.MD` to `README.md` and set the Week 0 docs back to mode 644.
- Pinned the repo to my profile; secret scanning and push protection are on.

## What broke and how I fixed it

1. **Two copies of Ansible.** apt had installed ansible 9.2.0 (core 2.16.3) next to the pipx copy, so the version I ran depended on PATH order. I removed the apt packages; `ansible --version` now points to `~/.local/bin`.
2. **VirtualBox refused the name `homelab-test`**, because the old VM still used it. The new VM is `test-homelab`, and that name is used everywhere: hostname, reservation, SSH alias and inventory.
3. **Pi-hole wouldn't save the DHCP reservation.** I typed the MAC address without colons; Pi-hole needs the `aa:bb:cc:dd:ee:ff` form.
4. **The root disk was 12 GB of a 25 GB disk.** The installer's default LVM layout left 11.5 GiB of the volume group unallocated. I grew it online with `sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv` (now 23G), then took the snapshot again. My first tries failed three ways: no `sudo`, no volume path, and `-lr`, which made `-l` read `r` as its size.
5. **Docs committed as executable.** The Week 0 docs were copied from `/mnt/c` and committed with mode 755. `chmod 644 docs/network/*.md` fixes that in this commit.
6. **A stale DHCP lease** for the old VM (`192.168.77.174`) was still in Pi-hole. Deleted.

## Evidence

- `ssh test-homelab hostname` returns `test-homelab` with no password prompt.
- The VM's ED25519 host key is `SHA256:CVqJTVnsyeaswTLnuwC0qVf6LmyRkJsndncdcbPWCvk` on the console and in WSL's `known_hosts` (`ssh-keygen -lF 192.168.77.50`).
- Root disk after the fix:

```text
$ df -h /
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   23G  5.4G   17G  25% /
```

## Notes for later weeks

- **Week 2, sudo-rs:** Ansible's `become` has a known problem with sudo-rs. The plan is to switch the VM to classic sudo with `update-alternatives` and record it in an ADR.
- **Week 3, SSH socket activation:** changing the SSH port needs `systemctl daemon-reload` and a restart of `ssh.socket`, not only `ssh`.
- **Week 6, port 53:** turn off `systemd-resolved`'s stub listener before Pi-hole starts.
- **Open question:** which device sends the IPv6 router advertisements (the `fda5:` prefix)?

## What I'd do differently

- Set the LVM volume to the full disk on the installer's storage screen instead of fixing it afterwards.
- Run `which -a ansible` before installing anything, to catch a duplicate install early.
