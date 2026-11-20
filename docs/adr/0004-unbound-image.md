# ADR 0004: Run Unbound from the klutchell/unbound image

**Status:** Accepted · **Date:** 20 Nov 2026

## Context

Week 6 runs Unbound as the recursive, DNSSEC-validating resolver behind Pi-hole, in a container deployed by Portainer from `stacks/dns/compose.yaml`. NLnet Labs, who make Unbound, don't publish an official Docker image, so the image has to come from a third party. v1 used `mvance/unbound`.

What the image has to offer:

- **A current Unbound release.** Unbound had a run of security releases in 2026. Every release before 1.26.1 has a critical heap overflow in its DNSSEC validator (CVE-2026-81642), and 1.26.2, released on 21 Oct 2026, fixes five more CVEs. A resolver is exposed to whatever answers come back from the internet, so anything older than 1.26.2 is out.
- **Version tags that can be pinned.** Same rule as Portainer: never `latest`, so a rebuild runs what was tested.
- **Active maintenance,** so the next security release arrives in days, not months.
- **A small attack surface:** no shell or package manager inside, and Unbound not running as root.
- **A documented way to add my own settings** without replacing the image's whole config, which already handles the trust anchor and root hints.

## Options considered

Checked on Docker Hub and GitHub on 20 Nov 2026.

| Image | Newest version tag | Last pushed | Notes |
| --- | --- | --- | --- |
| `mvance/unbound` (v1) | `1.22.0` | Over a year ago; GitHub repo last pushed October 2024 | Unmaintained. Every tag has the critical validator bug. ~92 MB Debian-based image. |
| `crazymax/unbound` | `1.26.1` | 2 months ago | Runs as non-root, listens on 5053, embeds the root trust anchor, overrides in `/config`. Hadn't picked up the 1.26.2 security release. |
| `madnuttah/unbound` | `1.26.1-1` | 20 days ago | Distroless, rootless option, optional healthcheck. Tags are `<unbound version>-<revision>`. Also not on 1.26.2 yet. |
| **`klutchell/unbound`** | **`1.26.2`** | **3 days ago** | Distroless, version tags, the only one already on the 1.26.2 security release. |

## Decision

Use `klutchell/unbound`, pinned by **tag and digest**:

```yaml
image: klutchell/unbound:1.26.2@sha256:de8e624f84337bdb2d16540c0b203d5df2a981180421cb50ab4212b67542d8b4
```

The tag is for humans; the digest makes Docker refuse anything else, even if someone re-pushes the `1.26.2` tag.

What the image looks like inside, from its Dockerfile and default config:

- **Built `FROM scratch`.** The build stages use Alpine 3.24.3, pinned by digest, and only the Unbound binaries and their libraries are copied into the final image. No shell, no package manager.
- **Drops root.** Unbound starts as root to bind port 53, then switches to the `unbound` user (UID 101, GID 102). Chroot is off, because there are no other files in the container to protect.
- **DNSSEC ready.** Root hints and the root trust anchor ship in `/var/unbound/`, and the trust anchor is kept up to date automatically (RFC 5011).
- **Listens on `0.0.0.0:53`** inside the container.
- **Loads custom files** with `include-toplevel: /etc/unbound/custom.conf.d/*.conf`, after its own settings, so they can override them.
- **Includes `unbound-checkconf` and `dig`,** so a config file can be tested before it's deployed.

**Config approach: an override file, not a full `unbound.conf`.** The image's own config keeps handling the trust anchor, root hints, user and chroot. The `dns_host` role templates only the settings from Pi-hole's Unbound guide into `/opt/homelab/unbound/pi-hole.conf`, and the stack mounts it read-only at `/etc/unbound/custom.conf.d/pi-hole.conf`. The guide's `interface`, `port` and `root-hints` lines are left out:

- `port: 5335` would move Unbound inside the container off 53, where the port mapping sends traffic.
- `interface: 127.0.0.1` inside a container is the container's own loopback, which traffic from Docker's port mapping never reaches.
- The image already ships root hints.

Unbound stays host-only through the port mapping instead: `127.0.0.1:5335:53`. The file also allows queries from `172.16.0.0/12`, the range Docker gives its networks, because Pi-hole's queries arrive through Docker's port mapping from the stack network, not from the container's own `127.0.0.1`. On the test VM that network is `172.18.0.0/16`, not Docker's default `172.17.0.0/16`, so a narrower rule would have refused Pi-hole.

**Pi-hole** uses the official `pihole/pihole` image, pinned to the date-style tag `2026.11.0`. There was nothing to choose there.

## Consequences

- **Updates are manual.** A new Unbound release means changing the tag and digest in `compose.yaml` and pushing. Portainer's polling then redeploys the stack. NLnet Labs' security advisories are the thing to watch.
- **One maintainer.** klutchell is a single person, which is how mvance went stale. If releases stop arriving within a week or two of NLnet Labs', switch to `madnuttah/unbound`, the runner-up. The override file is plain Unbound config, so it moves with only the mount path changing.
- **No shell in the container.** `docker exec unbound sh` doesn't work. Debugging goes through `docker logs unbound` and the image's own `dig`: `docker exec unbound dig @127.0.0.1 example.com`.
- **No healthcheck by default.** The image ships a `drill-hc` binary, but its Dockerfile doesn't use it, so `depends_on` only orders the start-up. Adding a healthcheck in the compose file is a stretch goal.
- **Config changes need a restart.** Unbound reads `pi-hole.conf` only when it starts. When Ansible changes the file, the container has to be restarted from Portainer.
- **Verified:** `unbound-checkconf` reported no errors with the override file mounted; `dnssec.works` resolves with the `ad` flag and `fail01.dnssec.works` returns `SERVFAIL` ([Week 6 build log](../build-log/week-06.md)).
