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
  config.yml             Reusable workflow: single source of truth for the
                         versions / platforms / latest_version constants
  ci.yml                 CI: smoke tests on PR and master push (full matrix)
  build.yml              Manual build test (workflow_dispatch, all versions x platforms)
  release.yml            Release: tag push (project release marker) or
                         workflow_dispatch rebuilds and republishes everything
  test-image.yml         Reusable workflow: functional smoke test via podman (amd64)
  build-image.yml        Reusable workflow: multi-arch build via buildx + QEMU,
                         optionally pushes to both registries (build test + release)
Makefile + scripts/test.mk
                         Local smoke test (`make test-build`) mirroring
                         test-image.yml
.context/
  optimize-ci.md         CI/CD architecture plan and rationale (read before
                         changing the workflows)
```

## Supported versions

- AlmaLinux **9.x**: `9.0`…`9.8`
- AlmaLinux **10.x**: `10.0`, `10.1`, `10.2`
- AlmaLinux **8.x** is **deprecated/archived** (see README "Archive tags").
  **Do not** re-add 8.x to the workflows or to the "supported" list.

Platforms built for every version: `linux/amd64`, `linux/arm64/v8`,
`linux/ppc64le`, `linux/s390x`.

## Local build and test

```sh
# Build a single version
docker build -t docker-almalinux-ansible . \
  --build-arg VERSION=10.2 \
  -f almalinux10/Dockerfile

# Functional smoke test (systemd + ansible)
podman build -t docker-almalinux-ansible:10.2 ./almalinux10/. --build-arg VERSION=10.2
podman run --name test-container --detach --privileged \
  --volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
  docker-almalinux-ansible:10.2
podman exec --tty test-container env TERM=xterm ansible --version
```

`ansible-core` is installed in the images; the Ansible binary may be invoked via
`ansible` or `ansible-playbook`. `sudo` is installed and `requiretty` is disabled.

Or via the Makefile: `make test-build` (override the version with
`make test-build version_branch=9.8`).

## GitHub Actions

Rules that must be respected when editing workflows:

- **All actions must be pinned by tag** (e.g. `actions/checkout@v7`) — **never**
  by commit SHA. Use floating major tags, not exact versions (`@v4`, not
  `@v4.6.0`). Dependabot keeps them up to date.
- Do **not** build or push Docker images during code changes to workflows —
  validation is done with `actionlint` + a YAML parser (see below). Images are
  only built by the pipelines themselves.
- Reusable workflows (`workflow_call`) do **not** inherit secrets: callers that
  need registry credentials must pass `secrets: inherit` (see the
  `build-image.yml` caller in `release.yml`). `GITHUB_TOKEN` is always
  available in the callee.
- GHCR login must use `GHCR_REGISTRY_TOKEN` (a PAT with `write:packages`), **not**
  `GITHUB_TOKEN`: the existing `kirill-zak/...` packages were created via that PAT,
  and a repo-scoped `GITHUB_TOKEN` is denied with `permission_denied: write_package`
  on pre-existing packages even with `permissions: packages: write`.
- `docker/build-push-action`: use a **narrowed context** (`./almalinuxN/.`)
  and do **not** set the `file` input — buildx then defaults to
  `<context>/Dockerfile`. Do **not** set `file: Dockerfile` explicitly: the
  `file` input resolves from the **repo root**, not the build context, and
  fails with "failed to read dockerfile" at build time.
- Tag triggers in `release.yml` use `tags: ['*']` — every tag push runs the
  workflow and `validate-tag` enforces the format (this avoids GitHub's glob
  dialect entirely). If ever using tag **patterns** again: `*` = any characters
  (not `/`), `+` = one or more of the preceding, `?` = zero or one; do **not**
  write `[0-9][0-9]*` for "one-or-more digits" (it means "two digits then
  anything") — use `[0-9]+`.

### Adding a new minor release (e.g. 10.2)

1. Add the version to `versions` in the "values" step of
   `.github/workflows/config.yml` (the single source of truth; every workflow
   drives its matrix from it).
2. If it is the newest version, update `latest_version` in
   `.github/workflows/config.yml` and the README tag list.
3. If it is the newest version, update the `version_branch` default in
   `scripts/test.mk`.

All matrices (CI, manual build, release) derive from `config.yml` outputs, so
no other workflow needs editing.

### Release process

- **Tag push** and **workflow_dispatch** behave identically: a full rebuild and
  republication of all image versions (12 tests + 12 multi-arch builds, with
  `latest` added as an extra tag for the newest version).
- Tags are **repository release markers** (e.g. `2.0.0`), **not** image versions.
  They must match `x.x.x` or `x.xx.xx` (each part 1–2 digits, optionally prefixed
  with `v`) — anything else fails loudly in the `validate-tag` job.
- The `latest` image is always built from `latest_version` in
  `.github/workflows/config.yml` (`release.yml` passes it as `additional_tags`).
- Reusable workflows called with `push: true` need registry credentials; the
  `release` job passes `secrets: inherit` to `build-image.yml`.

## Validation commands

```sh
# Lint all workflows (actionlint also runs shellcheck on inline scripts)
actionlint .github/workflows/*.yml

# Quick YAML syntax check
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release.yml'))"
```

## Conventions

- Branch names follow the GitHub issue pattern, e.g. `67-depricate-8x`.
- PR and commit messages are written in **English** and describe *why*; the
  README is the user-facing docs.
- Version lists in `README.md` (`Tags` and `Archive tags`) are arranged in
  descending order (newest first).
- Keep workflows DRY: prefer extending the reusable workflows
  (`test-image.yml`, `build-image.yml`, `config.yml`) over duplicating steps.
- `matrix` context is **not** available in job-level `if`; filter jobs by
  `inputs`/`github.event_name` only.
- The runtime stage of every `almalinux*/Dockerfile` must build on the **target**
  platform (`FROM --platform=$TARGETPLATFORM ...`), never `$BUILDPLATFORM` —
  otherwise every "multi-arch" variant ends up containing host-arch content
  while the CI only ever runs the amd64 variant.