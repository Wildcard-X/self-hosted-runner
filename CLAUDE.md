# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Dockerized GitHub Actions self-hosted runners. No application code, no test suite, no linter — the deliverables are two Docker images (`docker/linux` x64, `docker/mac` ARM64), their compose files, and `scripts/runner-pool.sh`. The README is the detailed user doc; keep it in sync with behavior changes.

## Commands

```sh
cp .env.example .env                                              # set REPO and REG_TOKEN (token expires in 1h)
docker build -t runner-test ./docker/linux                        # build locally (./docker/mac for ARM64)
docker build --build-arg RUNNER_VERSION=2.332.0 ./docker/linux    # override pinned runner version
docker compose -f docker/linux/docker-compose.yml up -d           # socket-mount (DooD) mode
docker compose -f docker/linux/docker-compose.dind.yml up -d      # Docker-in-Docker mode
./scripts/runner-pool.sh up 3 | status | upgrade | down | clean   # pool of N DinD stacks
bash -n docker/*/start.sh scripts/runner-pool.sh                  # syntax check
```

Compose files pull the GHCR image by default; uncomment `build: .` to use a local build. The only real test is spinning up a container and watching it register (`docker compose ... logs -f`).

## Architecture

**Two parallel variants that must stay in lockstep.** `docker/linux/` and `docker/mac/` each hold a `Dockerfile`, `start.sh`, `docker-compose.yml`, `docker-compose.dind.yml`, `dind-daemon.json`. They differ mainly in runner user/paths (`docker` + `/home/docker` on Linux, `runner` + `/home/runner` on ARM64), runner tarball arch, image tag (`latest` vs `latest-arm64`), and `platform: linux/arm64`. A fix to one almost always needs the same fix in the other (`diff docker/linux/start.sh docker/mac/start.sh` should show only the user/path lines). Known intentional divergence: only the Linux Dockerfile installs Playwright/Chromium shared libs.

**`start.sh` runs in two stages** (the container has no `USER` directive and starts as root):
1. As root: in DooD mode (no `DOCKER_HOST`, socket present), read the socket's GID, reuse/create a matching group, add the runner user as a *supplementary* member; chown `WORK_DIR`; then `exec setpriv --init-groups` re-running itself as the runner user. The re-exec is what makes the new group take effect.
2. As runner user: reset `HOME`/`USER` (setpriv leaves `HOME=/root`), wait up to 2 min for the daemon if `DOCKER_HOST` is set, `config.sh` (skipped if `.runner` already exists — the registration survives restarts past REG_TOKEN expiry), trap INT/TERM to deregister, `run.sh`. A non-zero `run.sh` exit deletes `.runner`/credentials so the next start re-registers.

**DinD mode wiring** (`docker-compose.dind.yml`) — several non-obvious constraints:
- Runner dials `tcp://docker:2376`, not `dind`: the sidecar's TLS cert is only valid for `docker`, hence the network alias.
- Runner uses `network_mode: service:dind` so ports published by job `services:` appear on the runner's `localhost`.
- `runner-work` volume is mounted at the *same absolute path* in both containers and `WORK_DIR` points at it, so `docker run -v $PWD:...` from a job resolves on the dind side. Don't let `.env` override `DOCKER_HOST`/`WORK_DIR` here.
- One replica per stack; scale by adding stacks (`-p runner-N`). `runner-pool.sh` discovers slots from compose project labels (`<prefix>-<n>`) and treats a slot as busy when `Runner.Worker` is running.

**NAME handling:** leave `NAME` unset so each replica registers under its unique container hostname; a fixed `NAME` makes replicas evict each other.

## Releases

- `RUNNER_VERSION` is an `ARG` in both Dockerfiles, bumped daily by `.github/workflows/update-runner-version.yml`, which also cuts tag `v<runner_version>` and calls the publish workflows directly (GITHUB_TOKEN tag pushes don't trigger workflows).
- Pushing any `v*` tag publishes both images to GHCR. Image-only changes get a human-cut revision tag: `v2.337.0-1`, `-2`, … (no `+`; must be a valid Docker tag).
- Never re-point an already-pushed tag — hosts with it cached won't re-pull.
