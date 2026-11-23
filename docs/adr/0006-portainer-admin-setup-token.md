# ADR 0006: Create Portainer's admin through the API with a custom setup token

**Status:** Accepted · **Date:** 23 Nov 2026

## Context

Week 8 makes `ansible-playbook playbooks/site.yml` take a bare VM to running stacks with no clicks. The first thing a fresh Portainer needs is an administrator account. Until Week 7 I created it by hand in the browser, inside Portainer's 5-minute first-run window.

Since 2.43 (and 2.39.4 on the older LTS line), a new Portainer instance with no administrator also requires a **setup token** before anyone can create the admin or restore a backup. Without it, someone else on the network could claim a freshly started instance first ([GHSA-x626-fcwx-f5pc](https://github.com/portainer/portainer/security/advisories/GHSA-x626-fcwx-f5pc)). By default the token is random and only printed in the container log, which a playbook can't read cleanly.

I checked how my pinned version, **2.45.3**, handles this in its source code (`api/cmd/portainer/main.go`, `api/http/handler/users/admin_init.go`):

- `--setup-token` (or the `PORTAINER_SETUP_TOKEN` environment variable) replaces the random token with my own value.
- The token is only used while **no administrator exists**. Once one does, Portainer ignores it.
- `POST /api/users/admin/init` reads the token from the **`X-Setup-Token` header**. It returns 200 on success, 403 for a missing or wrong token, and 409 if an admin already exists.
- `--admin-password-file` is read on **every** start. If the file is missing, Portainer refuses to start (`log.Fatal`).
- `GET /api/users/admin/check` returns 204 if an admin exists and 404 if not.

## Options considered

| Option | How it works | Secret left on the server | Downside |
| --- | --- | --- | --- |
| **A. Custom setup token + API** | Ansible sets `PORTAINER_SETUP_TOKEN` from Vault, then calls `admin/init` with the `X-Setup-Token` header | The token, visible in `docker inspect`, but useless once the admin exists | Must run within 5 minutes of the first start |
| B. `--admin-password-file` | Ansible writes the password to a file and mounts it; Portainer creates `admin` at start | The **real admin password, in plain text**, kept forever | Missing file means Portainer won't start |
| B2. `--admin-password` | Pass a bcrypt hash on the command line | The password hash, in `docker inspect` | A new random salt on every run recreates the container, unless the hash is stored in Vault |
| C. `--no-setup-token` | Turn the token check off | None | Brings back the problem the token was added to fix |

## Decision

**Option A.** The `portainer` role passes `PORTAINER_SETUP_TOKEN: "{{ vault_portainer_setup_token }}"` to the container. The new `portainer_stacks` role then:

1. Waits for Portainer to answer.
2. Calls `GET /api/users/admin/check`.
3. Only if that returns 404, calls `POST /api/users/admin/init` with `X-Setup-Token` and the username and password from Vault (`vault_portainer_admin_password`).

Every task that carries the token or the password uses `no_log: true`.

The admin password stays only in Vault and Bitwarden, never in a file on the server. The token only matters for the few seconds between Portainer's first start and Ansible's `admin/init`, and that's how this week's API work gets done.

## Consequences

- **Two new Vault variables** in `inventories/test/group_vars/all/vault.yml`: `vault_portainer_setup_token` (a long random value) and `vault_portainer_admin_password`. Prod gets its own in Week 9.
- **The Portainer container is recreated once.** Adding the environment variable changes the container's settings, so the first `site.yml` run after this change replaces it. Data is safe on the `portainer_data` volume, and the existing admin means the token is ignored.
- **The token is visible to anyone who can run `docker inspect` on the server.** That's root or the `docker` group, who could take over Portainer anyway through the Docker socket.
- **The 5-minute window still applies.** If Ansible fails between Portainer's first start and `admin/init`, Portainer disables admin creation after 5 minutes until it's restarted (`api/adminmonitor`). Re-running the playbook alone won't fix it, because the container is already running. Recovery: `docker restart portainer`, then re-run. The same token works again.
- **The Week 5 setup-timeout problem goes away** for normal runs, because no human has to race the window.
- **Portainer also needs `-H unix:///var/run/docker.sock`.** In 2.45.3 the Docker environment isn't created by the admin account itself: before Week 8, the browser's "Get started" step created it. With `-H`, Portainer creates an environment named `primary` right after `admin/init` (`api/internal/endpointutils/endpoint_setup.go`), and skips it when one already exists. The role looks the environment up by its URL, so it finds both `primary` on a fresh install and the older `local`.
- **Verified, 23 Nov 2026:** from `01-fresh-install`, `site.yml` created the admin (`changed`), then the environment, the Git source and both stacks, in 5 m 02 s. A second run skipped `admin/init` because `admin/check` returned 204. A `-vvv` run printed the setup token 0 times ([Week 8 build log](../build-log/week-08.md)).
