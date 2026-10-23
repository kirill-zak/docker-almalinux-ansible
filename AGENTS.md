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
  release.yml            Release: tag push (project release marker) or
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
- Reusable workflows (`workflow_call`) do **not** inherit secrets: callers that
  need registry credentials must pass `secrets: inherit` (see the
  `release-image.yml` callers in `release.yml`). `GITHUB_TOKEN` is always
  available in the callee.
- GHCR login must use `GHCR_REGISTRY_TOKEN` (a PAT with `write:packages`), **not**
  `GITHUB_TOKEN`: the existing `kirill-zak/...` packages were created via that PAT,
  and a repo-scoped `GITHUB_TOKEN` is denied with `permission_denied: write_package`
  on pre-existing packages even with `permissions: packages: write`.
- `docker/build-push-action` resolves the `file` input from the **repo root**,
  not from the build context. Use `context: ./` + `file: ./almalinuxN/Dockerfile`;
  a narrowed context (`./almalinuxN`) with `file: Dockerfile` fails with
  "failed to read dockerfile" at build time.
- Tag/branch triggers use GitHub's **own glob dialect**: `*` = any characters
  (not `/`), `+` = one or more of the preceding, `?` = zero or one. Do **not**
  write `[0-9][0-9]*` for "one-or-more digits" — it means "two digits then
  anything". Use `[0-9]+` instead (this kept `2.0.0` from triggering a release).

### Adding a new minor release (e.g. 10.2)

1. Add the version to the matrices in `.github/workflows/ci.yml`
   (the full list lives in the "Generate matrix" step of the `matrix` job;
   the PR version list is the two newest minors, update it too if needed).
2. Add it to `test-10` and `tag-release-branch-10` in
   `.github/workflows/release.yml`.
3. Add the minor to `test-build-10` matrix in `.github/workflows/build.yml`
   so the manual build test also covers it.
4. If it is the newest version, update `LATEST_VERSION` and the README tag list.

Keep the version lists in `ci.yml`, `build.yml`, and `release.yml` consistent —
every version in the release matrices is published on each release, so an
omission only shows up as a missing image tag.

### Release process

- **Tag push** and **workflow_dispatch** behave identically: a full rebuild and
  republication of all image versions (11 tests + 11 multi-arch builds + `latest`).
- Tags are **repository release markers** (e.g. `2.0.0`), **not** image versions.
  They must match `x.x.x` or `x.xx.xx` (each part 1–2 digits, optionally prefixed
  with `v`) — anything else fails loudly in the `validate-tag` job.
- The `latest` image is always built from `LATEST_VERSION`.

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
- The runtime stage of every `almalinux*/Dockerfile` must build on the **target**
  platform (`FROM --platform=$TARGETPLATFORM ...`), never `$BUILDPLATFORM` —
  otherwise every "multi-arch" variant ends up containing host-arch content
  while the CI only ever runs the amd64 variant.