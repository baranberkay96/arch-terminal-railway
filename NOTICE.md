# NOTICE: Arch Linux Web Terminal template

## Software installed in the image

| Component | Source | License | How it is used |
|---|---|---|---|
| Arch Linux | `archlinux` official image, https://hub.docker.com/_/archlinux, tag `base-20261108.0.623074`, index digest `sha256:d4594bb2…` (2026-11-10), upgraded with `pacman -Syu` at build | Various free licenses (see `/usr/share/licenses` in the image) | Base image (`FROM`) and pacman packages |
| ttyd 1.7.7 | Arch package `ttyd` (1.7.7, extra) | MIT, Copyright (c) 2016-2022 Shuanglei Tao | Web terminal on loopback, installed unmodified |
| nginx 1.30 | Arch package `nginx` | BSD-2-Clause | Basic-auth reverse proxy. The config was written for this template |
| tmux 3.8 | Arch package `tmux` | ISC | Persistent sessions |
| tini 0.19.0 | Static release binary from https://github.com/krallin/tini/releases/tag/v0.19.0, SHA-256 checked | MIT, Copyright (c) 2015 Thomas Orozco | Init and child subreaper, unmodified |
| Node.js 24 LTS | Arch package `nodejs-lts-krypton` and `npm` | MIT (bundled components under their own licenses) | Installed unmodified from the Arch repositories |
| base-devel, git, Python 3, ripgrep, fd, fastfetch and the other CLI tools | Arch packages | Each package's own license (GPL, MIT, BSD, Apache and similar) | Installed unmodified from the Arch repositories |

Changes to installed Arch files: file capabilities are removed from all binaries (Railway's containers cannot exec binaries whose capabilities they lack; this lets `sudo ping` work), the account `arch` (uid 1000, bash) is added, `/etc/sudoers.d/90-arch-terminal` grants it passwordless sudo, and `/etc/skel/.bashrc` gets one line that loads this template's shell defaults. nginx's own `nginx.conf` is replaced at every start by the file this template renders. No package files are edited otherwise.

Optional packages named in `ARCH_EXTRA_PACKAGES` are installed from the Arch repositories at build time under their own licenses.

Product names belong to their owners. This is a community template, not affiliated with Arch Linux, the ttyd author, the OpenJS Foundation or Railway.

## Upstream files and data checked (2026-11-20)

- Docker Hub tags of `library/archlinux`: `base-20261108.0.623074` (index `sha256:d4594bb2300cba11f41c2f27923eb56f63d9988909c402f9c95517d1e362c899`, amd64 manifest `sha256:fcee73ea…`, 134 MB compressed), the newest weekly `base` tag on 2026-11-20.
- Arch `core` and `extra` sync databases (x86_64, geo.mirror.pkgbuild.com): every package name above; `tini` and `dumb-init` are not in the official repositories, hence the release binary.
- https://github.com/krallin/tini/releases/tag/v0.19.0: `tini-static-amd64` and its `.sha256sum`.
- The ttyd behaviour this template relies on (auth-header mode `-H`, loopback `-i lo`, the `-c` startup log line that prints the credential) was checked for the Debian Web Terminal template of the same author against ttyd tag `1.7.7`; Arch ships the same release.

## Market templates reviewed

"Arch-Linux" (repo `XenaaLangliss/arch-railway`, ttyd with a typed password), "Arch Linux SSH" (bare `archlinux:latest` with `tail -f /dev/null`) and the "Linux Desktop | Arch Linux XFCE" webtop. Only their configuration and listing figures were used. The XenaaLangliss repository has no license, so it was read for ideas only; no files or text from other templates were used.

All files in this package were written for this template, adapted from the same author's Ubuntu, Debian and Kali Linux Web Terminal templates: the Dockerfile, `entrypoint.sh`, `terminal-session.sh`, `bashrc.sh`, `tmux.conf`, `nginx.conf.template`, `motd.txt` and the docs.
