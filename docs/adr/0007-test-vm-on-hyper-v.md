# ADR 0007: Run the test VM on Hyper-V instead of VirtualBox

**Status:** Accepted · **Date:** 25 Nov 2026

## Context

Every change runs on the test VM (`test-homelab`, 192.168.77.50) before it touches production, and each release is rebuilt on it from a bare Ubuntu install. That only works if a failed run on the test VM means my code is wrong, not that the VM is.

Until now the test VM ran in VirtualBox 7.2 on my Windows desktop. The desktop is also the Ansible control node, in WSL2, and WSL2 and Docker Desktop keep the Windows hypervisor switched on. VirtualBox then can't use the CPU's virtualization directly: it runs on top of Hyper-V through Microsoft's API, which it marks with a green turtle icon in the VM window. That mode is slower and less stable.

- In Weeks 7 and 8 the VM console showed `watchdog: BUG: soft lockup - CPU#0 stuck for 22s!` (up to 42 s) during several from-zero runs. Ansible and SSH stalled until it recovered, and once the VM was briefly unreachable (`No route to host`).
- A clean from-zero run needed several tries, so 5 m 02 s was the time of a run that didn't stall, not a typical one ([Week 8 build log](../build-log/week-08.md#test-environment-limitation-the-green-turtle-again)).
- The workarounds (shut the VM down after each session, never let Windows sleep with it running, 2 vCPUs, check `systemctl --failed` before trusting a result) made the stalls rarer, not gone.

Hyper-V wasn't an option on Windows 11 Home. On 24 Nov 2026 the desktop moved to Windows 11 Education, licensed through my college, which includes it.

## Options considered

| Option | How it works | Downside |
| --- | --- | --- |
| A. Keep VirtualBox with the workarounds | Nothing changes | The lockups stay, so a failed or slow run never clearly means a bug |
| B. Turn the Windows hypervisor off | `bcdedit /set hypervisorlaunchtype off` and a reboot, so VirtualBox runs natively | Breaks WSL2, which is my control node, and Docker Desktop. Switching back means another reboot every time |
| **C. Hyper-V** | Run the VM on the same hypervisor WSL2 already uses | Needs Pro or Education. New tool to learn. VirtualBox snapshots don't carry over, and the VM gets a new MAC address |
| D. A second physical machine | Test on bare metal, like production | I don't have a spare one |

## Decision

**Option C.** The test VM is rebuilt in Hyper-V with the same name, user, IP address and size:

| Setting | Value | Why |
| --- | --- | --- |
| Generation | 2 (UEFI) | Same firmware type as the ThinkPad |
| Secure Boot | On, template **Microsoft UEFI Certificate Authority** | The default template only trusts Windows; this one trusts Ubuntu's signed bootloader |
| CPU / memory / disk | 2 vCPUs, 4096 MB static (dynamic memory off), 25 GB VHDX | Same as the VirtualBox VM, so results compare |
| Network | External switch `External-LAN` on the wired adapter, sharing it with Windows | The VM sits on the home LAN like a real host. Bridging over Ethernet is reliable; Hyper-V's built-in Default Switch is NAT and would hide the VM from the LAN |
| MAC address | Static `00:15:5D:77:00:50` | Keeps the Pi-hole DHCP reservation for .50 valid. `00:15:5D` is Hyper-V's prefix; `77-00-50` reads as "LAN .77, host .50" |
| Checkpoints | **Standard**, automatic checkpoints off | Saves the running state like a VirtualBox snapshot. Production checkpoints would need Hyper-V's backup service inside Ubuntu, which isn't installed |
| Stop action | Shut down the guest | Windows shuts the VM down cleanly when the desktop restarts |

Checkpoints replace the snapshots: `01-fresh-install` (bare Ubuntu plus my SSH key), the starting point for from-zero runs, and `02-from-zero` (after `site.yml` and NPM's proxy hosts).

## Consequences

- **Faster and stable.** From `01-fresh-install`, `site.yml` ran in **2 m 05 s** (`changed=28`, `failed=0`) against 5 m 02 s on VirtualBox, with no stalls. A second run took 30 s with `changed=0`. The kernel log showed 0 soft lockups. The VirtualBox workarounds are no longer needed, though `systemctl --failed` before trusting a result stays a habit.
- **New MAC, same IP.** Production Pi-hole's reservation for 192.168.77.50 moved from the VirtualBox MAC `08:00:27:9A:9A:CF` to `00:15:5D:77:00:50` before the install, so the installer got .50 straight away.
- **New SSH host key.** The old key was removed with `ssh-keygen -R 192.168.77.50`, and the new fingerprint was checked on the VM console before accepting it.
- **NPM's proxy hosts were re-created by hand** (the [ADR 0005](0005-private-ca.md) gap). The `02-from-zero` checkpoint holds them until Week 9's backups.
- **Ubuntu's installer used only half the disk.** The default LVM layout gave the root volume 11 GB of 22 GB. `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` grew it to the full size before the first checkpoint. The ThinkPad install in Week 10 will show the same storage screen.
- **The desktop's own network now goes through the switch.** Windows uses `vEthernet (External-LAN)` instead of the physical adapter and kept its address. Creating the switch drops the network for a few seconds. The switch is tied to the wired adapter, so Wi-Fi isn't on it.
- **VirtualBox is gone.** It was uninstalled, its host-only network (192.168.56.0/24) no longer exists, and the old VM with its snapshots (29 GB) was deleted.
- **Production doesn't change.** The ThinkPad runs Ubuntu on bare metal; this only affects where changes are tested.
