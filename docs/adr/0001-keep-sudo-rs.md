# ADR 0001: Keep sudo-rs for Ansible privilege escalation

**Status:** Accepted · **Date:** 17 Nov 2026

## Context

Ubuntu 26.04 LTS replaced classic sudo (sudo.ws) with sudo-rs, a rewrite in Rust. Earlier Ansible releases could not recognize sudo-rs's password prompt, so any task with `become: true` hung and then failed with "Timed out waiting for become success or become password prompt" ([Ansible issue #85837](https://github.com/ansible/ansible/issues/85837), [Launchpad bug 2122414](https://bugs.launchpad.net/ubuntu/+source/ansible-core/+bug/2122414)). The common workaround was to switch the server back to classic sudo with `update-alternatives`.

Week 1 confirmed that the test VM uses sudo-rs (the `[sudo: authenticate] Password:` prompt), and the plan for Week 2 was to apply that workaround.

## What I tested

| Component | Version |
| --- | --- |
| Test VM | Ubuntu Server 26.04.1 LTS |
| sudo | sudo-rs `0.2.13-0ubuntu1.2` (`/usr/lib/cargo/bin/sudo` selected in `update-alternatives`) |
| Control node | ansible 14.6.0, ansible-core 2.21.6 (pipx, WSL Ubuntu 24.04) |

`ansible-playbook playbooks/bootstrap.yml -K --check --diff` with `become: true` completed with no timeout. The real runs, including one from a freshly restored snapshot, also worked. Ansible fixed the prompt handling upstream ([PR #86175](https://github.com/ansible/ansible/pull/86175)), and ansible-core 2.21.6 includes it.

## Options considered

1. **Switch to classic sudo with `update-alternatives`.** No longer needed. It would add a manual step before every fresh install, and Ubuntu is moving away from sudo.ws.
2. **Passwordless sudo for the Ansible user.** Avoids the password prompt entirely, but it is a security decision for the hardening work in Week 3, not a bug workaround.
3. **Keep sudo-rs, Ubuntu's default.** No extra steps, and the server stays on the configuration Ubuntu supports.

## Decision

Keep sudo-rs. No `update-alternatives` change and no manual pre-step in the README.

## Consequences

- The control node needs a recent ansible-core. Version 2.21.6 is tested; older versions may hang again. The README states the tested version.
- A related fix for translated (non-English) sudo-rs prompts is in [PR #86964](https://github.com/ansible/ansible/pull/86964). My servers use an English locale, so this doesn't affect them.
- **Rollback:** if `become` starts hanging after an Ansible or sudo-rs update, run `sudo update-alternatives --config sudo` on the server and pick sudo.ws. Then reopen this ADR.
