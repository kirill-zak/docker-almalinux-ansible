# Get version of AlmaLinux to build
ARG VERSION

FROM almalinux:${VERSION}
LABEL maintainer="Kirill Ziuzin"
ENV container=docker

RUN dnf -y update && dnf clean all

# Enable systemd.
RUN dnf -y install systemd && dnf clean all && \
  (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
  rm -f /lib/systemd/system/multi-user.target.wants/*;\
  rm -f /etc/systemd/system/*.wants/*;\
  rm -f /lib/systemd/system/local-fs.target.wants/*; \
  rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
  rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
  rm -f /lib/systemd/system/basic.target.wants/*;\
  rm -f /lib/systemd/system/anaconda.target.wants/*;

# Install requirements.
RUN dnf makecache \
  && dnf -y install \
    ansible-core \
    sudo \
    which \
  && dnf clean all

# Disable requiretty.
RUN sed -i -e 's/^\(Defaults\s*requiretty\)/#--- \1/'  /etc/sudoers

# Install Ansible inventory file.
RUN mkdir -p /etc/ansible
RUN echo -e '[local]\nlocalhost ansible_connection=local' > /etc/ansible/hosts

VOLUME ["/sys/fs/cgroup", "/tmp", "/run"]
CMD ["/usr/lib/systemd/systemd"]
