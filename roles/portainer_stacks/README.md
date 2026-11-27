# Role: portainer_stacks

Sets up Portainer through its API, so a fresh server needs no clicks in Portainer: creates the admin account on the first run, creates the Git source for this repo, and creates each Git-repository stack that doesn't exist yet. Portainer still owns the stacks once they exist ([ADR 0003](../../docs/adr/0003-ansible-owns-host-portainer-owns-apps.md)); this role only creates what's missing. How the admin is created is decided in [ADR 0006](../../docs/adr/0006-portainer-admin-setup-token.md).

## Requirements

- ansible-core 2.21 or later on the control node, with the `community.docker` collection (see `requirements.yml`).
- Privilege escalation: apply the role with `become: true` (it inspects the Portainer container).
- Apply it last, after `docker`, `dns_host` and `portainer`:
  - `portainer` must start Portainer with `PORTAINER_SETUP_TOKEN` and `-H unix:///var/run/docker.sock`.
  - `dns_host` must run **before** `portainer`. It switches the server's DNS away from systemd-resolved's stub listener; a Portainer container created before that switch keeps forwarding DNS to the stub, can't resolve `github.com`, and stack creation fails with HTTP 500.
- Portainer CE 2.45 (tested with 2.45.3 and 2.45.4). The API changed between releases: 2.45 uses Git sources (`SourceID`) and keeps the polling interval on the source.

## What it does

All tasks are in `tasks/main.yml`, in this order. The API calls run **on the server**, against the Portainer container's address on the `proxy` network (`https://<container IP>:9443/api`). Portainer publishes no ports, and on a fresh server Nginx Proxy Manager has no proxy hosts yet, so this is the only path that works from zero.

| Task | API call | What it does |
| --- | --- | --- |
| Check that the admin password is set | — | Stops early with a clear message if `portainer_stacks_admin_password` is empty. |
| Look up the Portainer container | — | `docker_container_info` on `portainer_stacks_container_name`. `no_log`, because the inspect output includes the container's environment, which holds the setup token. |
| Build the Portainer API address | — | Takes the container's IP on `portainer_stacks_network`. |
| Wait for Portainer to answer | `GET /system/status` | Retries until it returns 200 (up to 60 seconds). |
| Check whether an admin exists | `GET /users/admin/check` | 204 means an admin exists, 404 means it doesn't. Both count as success. |
| Create the admin user (first run only) | `POST /users/admin/init` | Only when the check returned 404. Sends the setup token in the `X-Setup-Token` header and the credentials from Vault. Reports `changed`. |
| Log in to the Portainer API | `POST /auth` | Gets a JWT. Never reports `changed`. |
| Save the auth header for later calls | — | `Authorization: Bearer <jwt>`, kept in a fact. |
| Wait for the Docker environment | `GET /endpoints` | Retries until an environment with URL `portainer_stacks_docker_url` exists. On a fresh install Portainer creates it (`primary`) just after the admin. |
| Save the environment ID | — | Found by URL, not by name: `local` when it came from the browser setup, `primary` when it came from `-H`. |
| List Git sources | `GET /gitops/sources` | |
| Create the Git source (first run only) | `POST /gitops/sources/git` | Only when no source is named `portainer_stacks_source_name`. Sets the URL and the polling interval. Returns 201. |
| Save the Git source ID | — | From the new source, or from the list. |
| List existing stacks | `GET /stacks` | |
| Save the existing stack names | — | `map(attribute='Name')`. |
| Create each missing stack | `POST /stacks/create/standalone/repository?endpointId=<id>` | One call per item in `portainer_stacks_list` whose name isn't taken, with `SourceID`, the branch, the compose path and the environment variables. Portainer clones the repo and starts the stack in the background. |

Every task that carries the password, the token or the JWT has `no_log: true`. Loop items show as `(censored due to no_log)` in the output. To debug a failing task, set `no_log: false` on that task only, for one run, and set it back.

The role has no handlers.

## Variables

All are in `defaults/main.yml`. The secrets are set from Vault in `inventories/<env>/group_vars/all/vars.yml`.

| Variable | Default | What it sets |
| --- | --- | --- |
| `portainer_stacks_container_name` | `portainer` | Container to inspect. Must match `portainer_container_name`. |
| `portainer_stacks_network` | `proxy` | Network whose IP the role uses to reach Portainer. Must match `portainer_network`. |
| `portainer_stacks_admin_user` | `admin` | Admin username. |
| `portainer_stacks_admin_password` | `""` | Admin password. Set it from Vault (`vault_portainer_admin_password`). Must meet Portainer's minimum length (12 characters by default). |
| `portainer_stacks_setup_token` | `""` | Must be the same value the `portainer` role passes as `portainer_setup_token` (both from `vault_portainer_setup_token`). |
| `portainer_stacks_docker_url` | `"unix:///var/run/docker.sock"` | URL of the Docker environment to deploy into. Must match `portainer_docker_endpoint`. |
| `portainer_stacks_source_name` | `homelab-as-code` | Name of the Git source. |
| `portainer_stacks_source_url` | `"https://github.com/R-Garren/homelab-as-code"` | Repository the stacks come from. Public, so no credentials. |
| `portainer_stacks_source_interval` | `5m` | How often Portainer polls the repo. At least `1m`. |
| `portainer_stacks_git_ref` | `refs/heads/main` | Branch every stack follows. |
| `portainer_stacks_list` | `[]` | Stacks to create. Each item has `name`, `compose_file` and `env` (a list of `name`/`value` pairs). Names are lowercase: Portainer normalizes them. |

## Example

```yaml
# inventories/test/group_vars/all/vars.yml
portainer_setup_token: "{{ vault_portainer_setup_token }}"
portainer_stacks_admin_password: "{{ vault_portainer_admin_password }}"
portainer_stacks_setup_token: "{{ vault_portainer_setup_token }}"

portainer_stacks_list:
  - name: dns
    compose_file: stacks/dns/compose.yaml
    env:
      - name: PIHOLE_PASSWORD
        value: "{{ vault_pihole_password }}"
      - name: HOMELAB_IP
        value: "{{ ansible_host }}"
  - name: proxy
    compose_file: stacks/proxy/compose.yaml
    env: []
```

```yaml
# playbooks/site.yml
- name: Configure homelab servers
  hosts: homelab
  become: true
  roles:
    - base
    - docker
    - dns_host
    - portainer
    - portainer_stacks
```

## Notes

- **It only creates, it never updates.** Changing an item's `env` in Vault or `vars.yml` doesn't change a stack that already exists; that needs a `PUT` to the stack (a stretch goal). Change it in Portainer, or delete the stack and re-run. Compose file changes don't need this role at all: Portainer polls the Git source.
- **`validate_certs: false`** on every call: Portainer's own certificate is self-signed, and the traffic never leaves the server (it runs over the `proxy` bridge). Behind NPM, the browser gets the private CA's certificate.
- **The 5-minute window.** If a run fails after Portainer's first start and before `admin/init`, Portainer disables admin creation after 5 minutes until it restarts. Re-running the playbook alone doesn't help, because the container is already running: run `docker restart portainer` on the server, then re-run.
- **Stacks deploy in the background.** The create call returns as soon as Portainer has cloned the repo; pulling images and starting containers takes another minute or two. Check `docker ps` after that, not right after the run.
- **Debugging without the role:** the same calls work by hand with `curl` and `jq`. The Week 8 build log has the exploration, including the status codes for each call.
- Nginx Proxy Manager's proxy hosts are still created by hand ([ADR 0005](../../docs/adr/0005-private-ca.md)). NPM has an API too.

## License

MIT
