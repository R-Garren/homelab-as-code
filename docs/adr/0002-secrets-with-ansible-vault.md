# ADR 0002: Keep secrets in Ansible Vault

**Status:** Accepted · **Date:** 18 Nov 2026

## Context

From Week 4 on, the playbooks need secrets: first the sudo (become) password of the server user, later the Pi-hole and Portainer admin passwords and the WireGuard keys. The repo is public, so a secret that reaches Git in plain text has to be treated as leaked.

Until now the sudo password was typed at the `-K` prompt on every run. That kept it off disk, but it doesn't scale to the secrets that Weeks 5–12 add, and nothing that runs unattended can type it.

In v1, secrets lived in `.env` files next to the Compose files, kept out of Git only by `.gitignore`.

## Options considered

1. **`.env` files (the v1 approach).** Simple, and Docker Compose reads them directly. But the secrets sit on disk in plain text, and one `git add -f`, a renamed file or a `.gitignore` typo publishes them. Ansible can't use them for its own variables, such as the become password.
2. **Passwordless sudo (`NOPASSWD`) for the Ansible user.** Removes the become password entirely, but then a stolen SSH key alone gives root on the server. This was rejected in Week 3 ([security baseline](../security-baseline.md), "No passwordless sudo"). It also does nothing for the application secrets.
3. **Ansible Vault.** Secrets are encrypted with AES-256 in a `vault.yml` file that can be committed safely. The vault password lives outside the repo. Built into Ansible, so there's nothing extra to install.

## Decision

Use Ansible Vault for every secret Ansible handles.

- Secrets live in `inventories/<env>/group_vars/all/vault.yml`, encrypted. Every file starts with `$ANSIBLE_VAULT;1.1;AES256`.
- Each secret's variable name starts with `vault_`. The plain `vars.yml` points to it, for example `ansible_become_password: "{{ vault_ansible_become_password }}"`. The variable names stay searchable with `grep`; the values stay encrypted.
- The vault password is a random 32-byte value (`openssl rand -base64 32`) in `~/.ansible/vault_pass.txt` on the control node, mode `600`, outside the repo. `ansible.cfg` points to it with `vault_password_file`, so no prompt appears.
- A copy of the vault password is kept in a password manager (Bitwarden), as a hidden field with master-password re-prompt.
- Vault files are edited with `ansible-vault edit` in a terminal editor (`nano`), not VS Code, so that editor history and backups can't keep a decrypted copy.

## Consequences

- `-K` is no longer needed. `ansible-playbook playbooks/site.yml` gets root on its own, and the second run is still `changed=0`.
- Becoming root still needs a password. Someone who steals only the SSH key, for example from a backup, can log in but can't become root. Someone who takes over my WSL user account gets both the SSH key and the vault password, and with them root on the server. This is accepted: the control node is my own desktop, and protecting it is part of the trust boundary.
- If the vault password file and the Bitwarden copy are both lost, the vault files can't be decrypted. The fix is to create new vault files and reset every secret in them.
- Before every commit, check that each `vault.yml` starts with `$ANSIBLE_VAULT`. `.gitignore` already blocks `vault_pass*` and `.vault_pass*`.
- **Rotation (documented, not yet run).** Rotate the vault password if it may have leaked, a device holding it is lost, or someone who knew it no longer should:
  1. `touch ~/.ansible/vault_pass.new && chmod 600 ~/.ansible/vault_pass.new`
  2. `openssl rand -base64 32 > ~/.ansible/vault_pass.new`
  3. `ansible-vault rekey --new-vault-password-file ~/.ansible/vault_pass.new <each vault.yml>`
  4. Update the Bitwarden copy.
  5. `mv ~/.ansible/vault_pass.new ~/.ansible/vault_pass.txt`, then confirm with `ansible-vault view` and a `site.yml` run.

  Rekeying changes the vault password only. If a secret inside the vault may have leaked, change that secret itself too.
- The production inventory (Week 10) gets its own `vault.yml`. Separate vault IDs per environment are an option if the test and prod passwords should differ.
