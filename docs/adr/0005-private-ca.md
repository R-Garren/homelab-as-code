# ADR 0005: Private CA for home.arpa, with an encrypted key kept outside Git

**Status:** Accepted · **Date:** 21 Nov 2026

## Context

Week 7 puts Portainer, Pi-hole and Nginx Proxy Manager (NPM) behind HTTPS at `portainer.home.arpa`, `pihole.home.arpa` and `npm.home.arpa`. Until now, Pi-hole's web UI has been plain HTTP, and Portainer has used its own self-signed certificate, which the browser warns about every time.

`home.arpa` is reserved for home networks (RFC 8375) and isn't a public domain, so no public CA like Let's Encrypt will issue certificates for it. That leaves three ways to get HTTPS:

- **Self-signed certificate per service.** Every service warns in the browser, and clicking through warnings is a bad habit to build.
- **A real domain with Let's Encrypt (DNS-01 challenge).** This costs money and puts a DNS provider API token on the server. It's worth a look after v2.0.
- **A private CA.** I sign my own certificates, and any machine that trusts my CA trusts them.

I chose the private CA. That makes the CA key the most sensitive file in the lab: whoever holds it can make certificates my desktop trusts. Most of this ADR is about where that key lives.

## Options considered for the CA key

| Option | Copies of the key | What protects it | If WSL is lost |
| --- | --- | --- | --- |
| A. Plain file outside Git, mode `0600` | One | File permissions only | The CA is gone; rebuild it and re-import it everywhere |
| B. Ansible Vault-encrypted key committed to the public repo | Every clone, and GitHub, forever | The vault password | Survives in Git |
| **C. Passphrase-encrypted file outside Git, backed up in Bitwarden** | **Two (WSL + Bitwarden)** | **File permissions and its own passphrase** | **Restore from Bitwarden** |

B was rejected. The encrypted key would be public forever: anyone could copy it and attack the vault password offline for as long as they like. Rekeying the vault doesn't help, because the old ciphertext stays in Git history.

## Decision

Option C:

- **Key location:** `~/.homelab-pki/` on the control node (WSL), folder `0700`, keys `0600`, outside the repo.
- **Key encryption:** PKCS#8 (`BEGIN ENCRYPTED PRIVATE KEY`), with a random 32-byte passphrase.
- **Passphrase:** stored in `playbooks/vars/pki/vault.yml` (variable `vault_pki_ca_passphrase`) and in Bitwarden. The pre-commit hook checks this file like every other `vault.yml`.
  - The vault file is public, but the passphrase alone is useless without the key, and the key never goes to Git.
  - It lives next to the playbook, not in `inventories/*/group_vars`, so only `playbooks/pki.yml` loads it and it isn't tied to one environment.
- **Backup:** one Bitwarden secure note holds the encrypted key and the passphrase.

The CA is built by `playbooks/pki.yml`, which runs on `localhost` only and uses `community.crypto`:

| | Root CA | NPM server certificate |
| --- | --- | --- |
| Key | ECC P-384, passphrase-encrypted | ECC P-256, no passphrase (NPM must read it) |
| Lifetime | 10 years | 397 days, the most browsers accept |
| Basic constraints | `CA:TRUE`, `pathlen:0`, critical | `CA:FALSE` |
| Key usage | `keyCertSign`, `cRLSign`, critical | Extended key usage `serverAuth` |
| Names | Name constraint `DNS:home.arpa`, critical | SANs `npm.home.arpa`, `pihole.home.arpa`, `portainer.home.arpa` |

- **Name constraint:** if the key ever leaks, it can only sign names under `home.arpa`, so it can't be used to fake `google.com` or my bank.
- **`pathlen:0`:** the CA can sign servers but not another CA.
- **Windows trust:** `ca.crt` goes into Windows' Trusted Root Certification Authorities store (current user). The Firefox-based browser trusts it through `security.enterprise_roots.enabled`.

**Finding:** without `format:`, `community.crypto.openssl_privatekey` wrote the CA key in OpenSSL's old format (`BEGIN EC PRIVATE KEY` with `Proc-Type: 4,ENCRYPTED`). That format turns the passphrase into the AES key with a single MD5 pass. `format: pkcs8` switches to PKCS#8, which derives the key with salted, iterated PBKDF2. The first key was deleted, and the CA was rebuilt before anything trusted it.

## Consequences

- **Renewal is manual.** The server certificate expires about 13 months after it's issued, and the playbook doesn't renew it on its own. To renew, re-issue the certificate and upload it to NPM again. The CA expires in November 2036.
- **No revocation.** There's no CRL or OCSP. If the server key leaks, I re-issue the certificate, and the 397-day lifetime limits how long the old one stays valid. If the CA key leaks, I remove the CA from Windows and build a new one; the name constraint limits the damage in the meantime.
- **Only devices I set up trust it.** Phones, laptops and guests still see warnings unless `ca.crt` is imported on them.
- **The passphrase is needed for every signing run.** Ansible reads it from Vault, so running `pki.yml` needs the vault password, the same as `site.yml`.
- **Losing WSL isn't fatal.** Restore `ca.key` from Bitwarden into `~/.homelab-pki/` with mode `0600`, then re-run the playbook.
- **Name constraints are a backstop.** Current Windows, Chrome and Firefox enforce them, and older clients that don't understand a critical extension must reject the certificate.

## Nginx Proxy Manager: where the certificate is used, and a known gap

NPM terminates HTTPS for all three names with the server certificate from `pki.yml`, uploaded by hand as a **custom certificate** (no intermediate: the CA is a root, and the desktop already trusts it). Its proxy hosts:

| Name | Forwards to | Why that target |
| --- | --- | --- |
| `npm.home.arpa` | `http://127.0.0.1:81` | NPM's own admin UI, inside the NPM container. Port 81 is published on loopback only. |
| `pihole.home.arpa` | `http://172.20.0.1:8081` | The host's address on the `proxy` network (`docker_proxy_gateway`). Pi-hole is on the host network; UFW allows 8081 only from `172.20.0.0/24`. |
| `portainer.home.arpa` | `https://portainer:9443` | Portainer's container name on the `proxy` network, with websockets on for its console. Nothing is published on 9443 any more. |

Every host has Force SSL and HTTP/2 on. HSTS is off: it would make browsers refuse plain HTTP for months, which is painful if the CA is ever rebuilt.

**Why `172.20.0.1` and not `host.docker.internal`:** NPM builds `proxy_pass` from a variable, so nginx resolves the name when a request arrives, through Docker's DNS (`127.0.0.11`). Docker's DNS knows container names but never reads the container's `/etc/hosts`, where `extra_hosts` writes `host.docker.internal`. The first try gave `502 Bad Gateway` with `host.docker.internal could not be resolved` in NPM's log, even though `getent` and `curl` inside the container resolved it. The gateway address comes from a subnet pinned in Ansible, so it doesn't depend on Docker's default bridge.

**The gap: NPM's configuration isn't in Git.** The proxy hosts and the uploaded certificate live in NPM's SQLite database in the `npm_data` volume, created by clicking in its UI. Everything else in this repo can be rebuilt from code; these three hosts can't. That breaks the "rebuildable with one command" goal for this one piece, and it was accepted for now because:

- **Caddy or Traefik would close it.** Both keep their routes in files (a `Caddyfile`, or labels and YAML), so the routes could live in `stacks/`. Switching means learning a second proxy and giving up NPM's UI, and it's worth a look after v2.0.
- **NPM has an API.** Week 8 already automates Portainer through its API; NPM's proxy hosts could be created the same way later.
- **Until then, backups cover it.** The VirtualBox snapshot `07d-proxy-hosts` held the test VM's NPM state; since the move to Hyper-V on 25 Nov 2026 the checkpoint `02-from-zero` does ([ADR 0007](0007-test-vm-on-hyper-v.md)). Week 9 backs up the `npm_data` volume and restores it in the disaster-recovery drill. The three hosts are also listed here and in the build log, so they can be re-created by hand in a few minutes.

**The certificate has to be re-uploaded** whenever `pki.yml` re-issues it (renewal, a new name). NPM keeps its own copy, so a new file in `~/.homelab-pki/` changes nothing until it's uploaded.
