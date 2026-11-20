# Arch Linux Web Terminal for Railway.
#
# Arch Linux (rolling) with ttyd serving tmux sessions, behind an nginx
# basic-auth proxy. The shell runs as the non-root user "arch" (passwordless
# sudo, base-devel for makepkg and AUR builds), and /home is a Railway volume.
# The base image is pinned by dated tag and digest; every build then runs a
# full `pacman -Syu`, so the system is as current as the Arch mirrors on the
# day it is built.
#
# The image is built in one stage and copied into an empty final stage, so it
# ships as a single layer. Railway's runtime showed directories and files that
# `pacman -Syu` had removed from the base layer (old /var/lib/pacman/local
# entries, so pacman reported "duplicated database entry"); a single layer
# carries no deletions over a lower layer.
FROM archlinux:base-20261108.0.623074@sha256:d4594bb2300cba11f41c2f27923eb56f63d9988909c402f9c95517d1e362c899 AS build

# tini static release binary (Arch has no tini package). The checksum comes
# from https://github.com/krallin/tini/releases/download/v0.19.0/tini-static-amd64.sha256sum.
ARG TINI_VERSION=0.19.0
ARG TINI_SHA256_AMD64=c5b0666b4cb676901f90dfcb37106783c5fe2077b04590973b885950611b30ee

# Extra Arch packages or groups to bake into the image, space separated, for
# example "go postgresql-libs". Railway passes the service variable of the
# same name as a build argument, so changing it rebuilds the image with those
# packages. They survive redeploys, unlike `pacman -S` in the shell.
ARG ARCH_EXTRA_PACKAGES=""

ENV LANG=C.UTF-8 \
    TZ=UTC

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Full system upgrade plus the auth proxy (nginx), the web terminal (ttyd
# 1.7.7 from extra), tmux, base-devel (gcc, make, sudo, makepkg) and the tools
# people expect in a shell. Node.js comes from the 24 LTS "Krypton" package.
# The Arch image keeps no man pages or docs (NoExtract in pacman.conf). The
# package cache is emptied; the sync databases stay so `pacman -Ss` works.
RUN pacman -Syu --noconfirm --needed \
      base-devel bash-completion bind curl fastfetch fd git git-lfs htop iproute2 \
      iputils jq less libcap lsof nano ncdu nginx nodejs-lts-krypton npm \
      openbsd-netcat openssh openssl procps-ng python python-pip python-pipx \
      ripgrep rsync sqlite tmux tree ttyd tzdata unzip vim wget which xz zip \
 && rm -rf /var/cache/pacman/pkg/* \
 && test -z "$(pacman -Qq 2>&1 > /dev/null)" \
 && ttyd --version \
 && node --version \
 && npm --version

# Optional extras from ARCH_EXTRA_PACKAGES, checked as package names first so
# the variable cannot inject shell or pacman options. A value typed into the
# deploy form must not fail the build: invalid names and packages that pacman
# cannot install are skipped with a warning (here and in the deploy log, from
# /etc/arch-terminal/skipped-packages), and the rest is installed.
RUN set -eux; \
    install -d -m 0755 /etc/arch-terminal; \
    : > /etc/arch-terminal/skipped-packages; \
    read -r -a requested <<< "${ARCH_EXTRA_PACKAGES//,/ }"; \
    extra=(); \
    for pkg in "${requested[@]}"; do \
      if [[ "$pkg" =~ ^[a-z0-9@_+][a-z0-9@._+-]*$ ]]; then \
        extra+=("$pkg"); \
      else \
        echo "WARNING: ARCH_EXTRA_PACKAGES: skipping invalid package name '${pkg:0:64}'" >&2; \
        printf '%s\n' "${pkg:0:64}" >> /etc/arch-terminal/skipped-packages; \
      fi; \
    done; \
    if (( ${#extra[@]} > 0 )); then \
      if ! pacman -S --noconfirm --needed "${extra[@]}"; then \
        echo "WARNING: ARCH_EXTRA_PACKAGES: installing '${extra[*]}' together failed. Trying one package at a time." >&2; \
        for pkg in "${extra[@]}"; do \
          if ! pacman -S --noconfirm --needed "$pkg"; then \
            echo "WARNING: ARCH_EXTRA_PACKAGES: skipping '${pkg}', pacman could not install it." >&2; \
            printf '%s\n' "$pkg" >> /etc/arch-terminal/skipped-packages; \
          fi; \
        done; \
      fi; \
      rm -rf /var/cache/pacman/pkg/*; \
    fi; \
    chmod 0644 /etc/arch-terminal/skipped-packages

RUN set -eux; \
    curl -fsSL --retry 5 --retry-all-errors --retry-delay 2 -o /usr/local/bin/tini \
      "https://github.com/krallin/tini/releases/download/v${TINI_VERSION}/tini-static-amd64"; \
    echo "${TINI_SHA256_AMD64}  /usr/local/bin/tini" | sha256sum -c -; \
    chmod 0755 /usr/local/bin/tini; \
    /usr/local/bin/tini --version

# Railway's container capability set lacks NET_RAW, and the kernel refuses to
# exec a binary whose file capabilities exceed that set: Arch's ping
# (cap_net_raw) fails with "Operation not permitted" for every user, root
# included. Without file capabilities ping falls back to ICMP datagram
# sockets, which work under sudo.
RUN set -eux; \
    caps="$(getcap -r /usr 2> /dev/null || true)"; \
    for bin in $(printf '%s\n' "$caps" | awk 'NF {print $1}'); do setcap -r "$bin"; done; \
    test -z "$(getcap -r /usr 2> /dev/null || true)"

# The login user "arch" (uid 1000, bash, passwordless sudo). Its home is
# created on the volume at first boot from /etc/skel: Arch's default dotfiles
# plus one line in .bashrc that loads this template's shell defaults.
RUN set -eux; \
    printf '\n# Arch Linux Web Terminal defaults (prompt, history, PATH). Delete these lines to opt out.\nif [ -r /etc/arch-terminal/bashrc ]; then . /etc/arch-terminal/bashrc; fi\n' \
      >> /etc/skel/.bashrc; \
    useradd --uid 1000 --user-group --shell /bin/bash --no-create-home --home-dir /home/arch arch; \
    rm -rf /home/arch; \
    printf 'arch ALL=(ALL:ALL) NOPASSWD: ALL\n' > /etc/sudoers.d/90-arch-terminal; \
    chmod 0440 /etc/sudoers.d/90-arch-terminal; \
    visudo -cf /etc/sudoers.d/90-arch-terminal

COPY entrypoint.sh /usr/local/bin/arch-terminal-entrypoint
COPY terminal-session.sh /usr/local/bin/terminal-session
COPY bashrc.sh /etc/arch-terminal/bashrc
COPY motd.txt /etc/arch-terminal/motd
COPY nginx.conf.template /etc/arch-terminal/nginx.conf.template
COPY tmux.conf /etc/tmux.conf
RUN chmod 0755 /usr/local/bin/arch-terminal-entrypoint /usr/local/bin/terminal-session \
 && chmod 0644 /etc/arch-terminal/bashrc /etc/arch-terminal/motd \
      /etc/arch-terminal/nginx.conf.template /etc/tmux.conf

FROM scratch
COPY --from=build / /

ENV LANG=C.UTF-8 \
    TZ=UTC \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/bin:/usr/bin/site_perl:/usr/bin/vendor_perl:/usr/bin/core_perl

EXPOSE 8080

# On Railway, podman-init is PID 1 and this runs under it. tini -s registers as
# a child subreaper, so orphaned processes (the tmux server, background jobs)
# are reaped whether or not tini is PID 1.
ENTRYPOINT ["/usr/local/bin/tini", "-s", "--", "/usr/local/bin/arch-terminal-entrypoint"]
