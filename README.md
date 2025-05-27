# docker-almalinux-ansible
AlmaLinux docker container with built-in Ansible for playbook and role testing 

## Tags
- `8.x`: Based on branch `8.x`. Releases: `8.5`, `8.6`, `8.7`, `8.8`, `8.9`, `9.10`.
- `9.x`: Based on branch `9.x`. Releases: `9.0`, `9.1`, `9.2`, `9.3`, `9.4`, `9.5`, `9.6`.

## Platforms
- `linux/amd64`
- `linux/arm64/v8`
- `linux/ppc64le`
- `linux/s390x` (except `8.5`)

## How to Build

To build the image on your own locally, do the following:

  1. [Install Docker](https://docs.docker.com/engine/installation/).
  2. `cd` into target directory. For example, target directory for `AlmaLinux 9.x` is `almalinux9`
  3. Run `docker build -t docker-almalinux-ansible . --build-arg VERSION=9.<release>`. Where `<release>` is release version of  AlmaLinux. List of minor releases can found on official release page of AlmaLinux (https://wiki.almalinux.org/release-notes/)

  ## How to Use

  1. [Install Docker](https://docs.docker.com/engine/installation/).
  2. Pull pre-build image via `docker image pull kirillzak/docker-almalinux-ansible:latest` or build image.
  3. Run a container from the image: `docker run --detach --privileged --volume /sys/fs/cgroup:/sys/fs/cgroup:ro docker-almalinux-ansible:latest`

## Author

[Kirill Ziuzin](https://kirill-zak.ru/)