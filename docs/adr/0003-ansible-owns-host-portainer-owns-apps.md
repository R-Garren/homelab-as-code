# ADR 0003: Ansible owns the host, Portainer owns the apps

**Status:** Accepted · **Date:** 18 Nov 2026

## Context

In v1, Ansible deployed the app stacks by running `docker compose` on the server. Portainer showed each of them as "This stack was created outside of Portainer. Control over this stack is limited.": no editor and no redeploy, only start and stop.

v2 wants two things at once: everything lives in Git and can be rebuilt, and Portainer has full control over the apps it shows.

In Week 5 the v1 problem was reproduced on purpose on the test VM, with two copies of a one-container `traefik/whoami` stack:

| Stack | Created with | What Portainer showed |
| --- | --- | --- |
| `whoami-cli` | `docker compose up -d` in a shell | The "limited" banner, no Editor tab. Its page URL ends in `external=true` and has no stack ID. |
| `whoami-portainer` | Stacks → Add stack → Web editor | Stack and Editor tabs, Stop/Delete buttons. Its URL carries `id=1&regular=true`. |

Screenshots: [limited control](../img/week-05-stack-limited-control.png), [full control](../img/week-05-stack-full-control.png).

**Root cause:** Portainer fully manages only stacks it has a record and a compose file for in its own database, which it keeps under `/data/compose/<stack id>` in its volume. A stack started anywhere else is only recognised from the `com.docker.compose.project` label that Compose puts on its containers, so Portainer has nothing to edit or redeploy.

## Options considered

| Option | Portainer control | Lives in Git? | Notes |
| --- | --- | --- | --- |
| Ansible runs `docker compose` | Limited (v1's problem) | Yes | Ansible-only workflow. Portainer is reduced to a viewer. |
| Portainer web editor | Full | No | Click-ops: the compose file lives only in Portainer's database, so a lost volume means lost stacks. Not infrastructure as code. |
| **Portainer Git-repository stacks** | Full | Yes | Portainer clones the repo and deploys `stacks/<name>/compose.yaml`. It keeps its own record, so it has full control. |

## Decision

Split ownership at the Docker Engine:

- **Ansible owns the host.** The OS, hardening and firewall (`base`), Docker Engine (`docker`), Portainer itself (`portainer`), and any config file an app needs. Ansible places those config files under `/opt/homelab/<stack>/`.
- **Portainer owns the app stacks.** Each one is created in Portainer as a Git-repository stack pointing at `stacks/<name>/compose.yaml` in this repo. Changes go through Git, and then the stack is redeployed from Portainer.
- Nothing is deployed with `docker compose` from a shell or from Ansible. The web editor is only for throwaway tests that are deleted afterwards, like the Week 5 test.
- Portainer itself is a single container run by `community.docker.docker_container`, not a Compose stack, so this decision doesn't apply to it.

## Consequences

- Portainer has full control over every app stack, and Git stays the source of truth for what each stack runs.
- **Bind mounts use absolute paths.** Portainer CE doesn't support relative paths for Git stacks: the "Enable relative path volumes" option is Business Edition only. CE clones the repo inside its own container under `/data/compose/<stack id>`, so `./config` would point at a folder that doesn't exist on the host. Compose files therefore mount `/opt/homelab/<stack>/...`, and Ansible must create those files before the stack is deployed. Business Edition wouldn't remove this rule either: users report that stacks deployed with relative paths show as "Limited control", because a helper container creates them (GitHub discussion #9795).
- **Creating a stack is a manual UI step until Week 8,** when Ansible creates the Git stacks through Portainer's API. Until then, a rebuild from scratch means re-adding each stack by hand.
- **Portainer's volume holds the stack records.** A lost `portainer_data` volume means re-creating the stacks, but not losing their contents, since the compose files are in Git. Week 9 adds backups.
- **Portainer CE stays on the 2.45 LTS line.** In September 2026 Portainer announced that there will be no CE build of 3.x; 2.45 LTS keeps getting security and bug fixes. The compose files in `stacks/` are plain Compose and don't depend on Portainer. If CE stops being maintained, they can move to another manager, such as Komodo, without rewriting the apps.
- **Portainer is a high-value target.** It mounts the Docker socket, so a Portainer admin is effectively root on the host. The controls are listed in the [security baseline](../security-baseline.md): a strong admin password, and port 9443 LAN-facing only until Week 7, when Nginx Proxy Manager fronts it.
