# Arch Linux Web Terminal (Browser Shell): Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Arch Linux Web Terminal (Browser Shell)** (no version label: Arch is a rolling release and every build runs `pacman -Syu`) |
| Description | Persistent Arch Linux shell in your browser, password-protected, AUR-ready. (75 chars) |
| Category | Other |
| Tags | arch, terminal, web-terminal, linux, shell (the first five are sent), archlinux, ttyd, tmux, aur |
| Icon | `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/arch-linux.svg` (200, image/svg+xml) |

Why: the Arch niche had about 48 deploys in 30 days (marketplace data, 2026-11). "Arch-Linux" (XenaaLangliss) runs ttyd with a typed password and no volume; "Arch Linux SSH" is a bare `archlinux:latest` kept alive with `tail -f /dev/null`. This package brings our Ubuntu/Debian/Kali web terminal design (116, 60, 109) to Arch.

## Base

Derived from `116-ubuntu-terminal` and `127-alpine-terminal` (same nginx auth proxy, `/healthz`, ttyd auth-header mode, tmux sessions, fail-closed password handling per STANDARD 8.29, build-time extra packages like 109's `KALI_EXTRA_PACKAGES`).

Arch-specific changes:
- **Single-layer image.** The first test build showed 26 `duplicated database entry` errors from pacman at runtime: the old `/var/lib/pacman/local/<pkg>-<oldver>` directories (and 15 old library files) that `pacman -Syu` had removed from the base layer were visible again in the running container, while the build itself had no duplicates. The Dockerfile now builds in a `build` stage and copies `/` into `FROM scratch`, so the shipped image has one layer and no deletions over a lower layer. The build also fails if `pacman -Qq` reports any error.
- `pacman -Syu` on every build; the sync databases stay in the image so `pacman -Ss` works. Users should install with `sudo pacman -Syu PACKAGE` (no partial upgrades).
- `base-devel` is included, and the user `arch` is a normal user with sudo, so `makepkg -si` and AUR builds work.
- Node.js from `nodejs-lts-krypton` (24 LTS); tini from the static GitHub release with SHA-256 (no Arch package).
- nginx runs as `http`; temporary paths in `/run/terminal-nginx`; nginx starts with `-e stderr`.

## Pinned versions (checked 2026-11-20)

| Component | Source | Version |
|---|---|---|
| Base image | `archlinux:base-20261108.0.623074@sha256:d4594bb2300cba11f41c2f27923eb56f63d9988909c402f9c95517d1e362c899` | weekly base image, then `pacman -Syu` |
| ttyd | Arch extra | 1.7.7-4 |
| Node.js / npm | `nodejs-lts-krypton` / `npm` | 24.22.3 / 12.2.3 (at build time) |
| nginx | Arch extra | 1.30.7 |
| tmux | Arch extra | 3.8 |
| tini | GitHub release `tini-static-amd64`, SHA-256 checked | 0.19.0 |

Package versions other than tini move with the Arch repositories (rolling release).

## Services

### Terminal

| Setting | Value |
|---|---|
| Source | repo `baranberkay96/arch-terminal-railway`, root `/services/terminal` (Dockerfile) |
| Healthcheck | `/healthz` on `PORT=8080`, timeout 120 s |
| Public domain | yes, target port 8080 |
| Volume | `/home` (home of `arch` is `/home/arch`) |
| Restart policy | ON_FAILURE, 10 retries |
| Replicas | 1 |

### Variables

| Variable | Default | Notes |
|---|---|---|
| `PASSWORD` | `${{secret(32)}}` | 12-256 bytes, one line. Otherwise the login is locked with a random password; the deploy still succeeds and the log says how to fix it. |
| `USERNAME` | `arch` | Invalid names fall back to `arch` with a warning. |
| `PORT` | `8080` | Invalid values or 7681 fall back to 8080 with a warning. |
| `TERMINAL_TMUX` | `true` | `false`: plain login shell per tab. |
| `TERMINAL_FONT_SIZE` | `15` | 8-40; anything else falls back to 15 with a warning. |
| `ARCH_EXTRA_PACKAGES` | empty | Build argument. Packages or groups from the official repos. Invalid names and packages pacman cannot install are skipped with a warning (build log and deploy log); never fails the build. |
| `TZ` | `UTC` | Shell time zone. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `15` | Grace period for the clean tmux shutdown. |

No required inputs.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle (measured) | Typical | Notes |
|---|---|---|---|
| Terminal | 0.034 GB RAM, ~0 vCPU (measured, 0.5 GB limit) | 0.18 GB right after `npm install -g` incl. page cache (measured), 0.3 GB avg | `makepkg` and compiles peak higher |
| Volume `/home` | 0.1 GB | 2-5 GB | Projects, AUR build dirs, venvs |

Estimated monthly cost: idle about $0.50, typical about $4. Minimum plan: Free (0.5 GB RAM per service). Image: about 240 packages, about 1.2-1.5 GB installed, about 0.45 GB compressed as one layer (estimate from the sync databases), below the 4 GB Trial/Free cap.

## Improvements over market templates

1. Generated `secret(32)` password, nginx basic auth with a SHA-512 crypt hash, rate limit, cross-site WebSocket check, loopback-only ttyd; the password never reaches logs (ttyd `-c` prints it). Fails closed without failing the deploy.
2. The whole `/home` persists (dotfiles, history, AUR build dirs, venvs, pipx, `npm -g`). Market: no volume, or `/data` only.
3. tmux sessions survive closed tabs; `?arg=NAME` opens separate sessions.
4. A non-root user with sudo and base-devel, so AUR builds work (makepkg refuses to run as root).
5. Pinned base image plus `pacman -Syu` at build, single-layer image without stale pacman entries.
6. `/healthz` healthcheck with `PORT` set; build-time extra packages that never fail the build.

## Verification checklist

Done on the pre-publish test deploy (2026-11-20):

- [x] First build: deploy SUCCESS but `pacman -Q` printed 26 "duplicated database entry" errors (see Base). Fixed with the single-layer build; after the fix 0 duplicates, 0 stale files, `pacman -Qq` clean.
- [x] `/healthz` 200; `/` 401 without/with a wrong password, 200 with `PASSWORD`; WebSocket 101 with the password, 403 with a foreign Origin, 401 with a wrong password.
- [x] Shell: `whoami` = `arch`, `sudo -n true`, node v24.22.3, Python 3.14.10, GCC 16.2.1, `sudo ping`, git prompt loaded, `PASSWORD` not in the environment, home 0750.
- [x] AUR: `git clone https://aur.archlinux.org/yay-bin.git && makepkg -s` built `yay-bin-13.1.0-1`.
- [x] `~/persist.txt` survived three redeploys (including the switch to the single-layer image).
- [x] `PASSWORD=abc`: SUCCESS, lock warning, every password 401. `USERNAME="bad user!"`, `TERMINAL_FONT_SIZE=abc`: SUCCESS with fallback warnings.
- [x] 0.5 GB limit: `memory.max` 500 MB, 0.034 GB idle.
- [ ] Publish + post-publish test (blocked: workspace billing).

## Open questions

- Every rebuild pulls the current Arch repositories; an upstream packaging problem on that day would fail the build (rolling-release risk). The base image digest stays pinned.
