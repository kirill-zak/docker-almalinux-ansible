# AGENTS.md

Guidance for AI agents and contributors working on this repository.

## Project overview

This repository builds Docker images of AlmaLinux with a pre-installed Ansible,
intended for testing Ansible playbooks and roles in containers. Images run
**systemd as PID 1**, so they must be started with `--privileged` and the
cgroup filesystem mounted (`-v /sys/fs/cgroup:/sys/fs/cgroup:ro`).

Images are published to:
- Docker Hub: `kirillzak/docker-almalinux-ansible`
- GHCR: `ghcr.io/kirill-zak/docker-almalinux-ansible`

## Repository structure

```
almalinux9/
almalinux10/     One directory per AlmaLinux major version, each with a single
                 Dockerfile (parameterized via ARG VERSION, e.g. 9.7)
.github/workflows/
  ci.yml                 CI: smoke tests on PR (latest minors) and master push (full matrix)
  build.yml              Manual build test (workflow_dispatch, all 4 platforms)
  release.yml            Release: tag pushes publish only the tagged version;
                         workflow_dispatch rebuilds and republishes everything
  test-image.yml         Reusable workflow: functional smoke test via podman (amd64)
  build-image.yml        Reusable workflow: multi-arch build test via buildx + QEMU
  release-image.yml      Reusable workflow: multi-arch build & push to both registries
```

## Supported versions

- AlmaLinux **9.x**: `9.0`…`9.8`
- AlmaLinux **10.x**: `10.0`, `10.1`
- AlmaLinux **8.x** is **deprecated/archived** (see README "Archive tags").
  **Do not** re-add 8.x to the workflows or to the "supported" list.

Platforms built for every version: `linux/amd64`, `linux/arm64/v8`,
`linux/ppc64le`, `linux/s390x`.

## Local build and test

```sh
# Build a single version
docker build -t docker-almalinux-ansible . \
  --build-arg VERSION=10.1 \
  -f almalinux10/Dockerfile

# Functional smoke test (systemd + ansible)
podman build -t docker-almalinux-ansible:10.1 ./almalinux10/. --build-arg VERSION=10.1
podman run --name test-container --detach --privileged \
  --volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
  docker-almalinux-ansible:10.1
podman exec --tty test-container env TERM=xterm ansible --version
```

`ansible-core` is installed in the images; the Ansible binary may be invoked via
`ansible` or `ansible-playbook`. `sudo` is installed and `requiretty` is disabled.

## GitHub Actions

Rules that must be respected when editing workflows:

- **All actions must be pinned by tag** (e.g. `actions/checkout@v7`) — **never**
  by commit SHA. Use floating major tags, not exact versions (`@v4`, not
  `@v4.6.0`). Dependabot keeps them up to date.
- Do **not** build or push Docker images during code changes to workflows —
  validation is done with `actionlint` + a YAML parser (see below). Images are
  only built by the pipelines themselves.

### Adding a new minor release (e.g. 10.2)

1. Add the version to the matrices in `.github/workflows/ci.yml`
   (the full list lives in the "Generate matrix" step of the `matrix` job;
   the PR version list is the two newest minors, update it too if needed).
2. Add it to `test-10` in `.github/workflows/release.yml`.
3. Add it to the `supported` list in the `validate-tag` job of
   `.github/workflows/release.yml` (a tag it does not recognize fails loudly,
   so keeping the list in sync is mandatory).
4. Add the minor to `test-build-10` matrix in `.github/workflows/build.yml`
   so the manual build test also covers it.
5. If it is the newest version, update `LATEST_VERSION` and the README tag list.

Keep the version list in `ci.yml`, `build.yml`, `release.yml`, and `test-image.yml`
callers consistent — `validate-tag` in `release.yml` is the guard that catches drift.

### Release process

- **Tag push** (e.g. `10.1.0`): validates the tag (`MAJOR.MINOR.PATCH`, each
  part 1–2 digits, optionally prefixed with `v`), tests and publishes **only**
  the tagged version. The published image version is `MAJOR.MINOR` (the patch
  distinguishes re-releases of the same image), and the image tag is refreshed
  only if that version is not older than `LATEST_VERSION`.
- **workflow_dispatch**: full rebuild and republication of all versions
  (11 tests + 11 multi-arch builds + `latest`).
- Tags whose `MAJOR.MINOR` is not in the matrices (e.g. legacy `1.0.16` →
  `1.0`) are rejected by `validate-tag`.

## Validation commands

```sh
# Lint all workflows (actionlint also runs shellcheck on inline scripts)
actionlint .github/workflows/*.yml

# Quick YAML syntax check
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release.yml'))"
```

## Conventions

- Branch names follow the GitHub issue pattern, e.g. `67-depricate-8x`.
- PR/commit messages describe *why*; the README is the user-facing docs.
- Keep workflows DRY: prefer extending the reusable workflows
  (`test-image.yml`, `build-image.yml`, `release-image.yml`) over duplicating steps.
- `matrix` context is **not** available in job-level `if`; filter jobs by
  `inputs`/`github.event_name` only.