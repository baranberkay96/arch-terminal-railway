# Deploy and Host Arch Linux Web Terminal with Railway

An Arch Linux shell that opens in any browser. You work as the user `arch` with sudo, inside tmux, so sessions and long-running jobs survive closed tabs. A generated password protects the login, and your home directory sits on a persistent volume. base-devel, Git, Python, Node.js 24 and everyday CLI tools come preinstalled, so `makepkg` and AUR builds work out of the box.

## About Hosting Arch Linux Web Terminal

A bare `archlinux` container on Railway has no way in except Railway's own CLI. This community template turns it into a working terminal: ttyd serves the shell over HTTPS, nginx stands in front of it and checks a password that Railway generates at deploy, and tmux keeps your work running between visits. The image builds on the official `archlinux` image from Docker Hub and runs a full `pacman -Syu` on every build, so each deploy is as current as the Arch mirrors. `/home` is a Railway volume, Railway's edge provides HTTPS, and the container restarts automatically if it stops.

## Common Use Cases

- A rolling-release Linux shell in the cloud for Chromebooks, tablets, locked-down laptops or any machine without SSH
- Building and testing AUR packages and PKGBUILDs in a clean, disposable Arch environment
- Trying the newest compilers, runtimes and CLI tools before they reach other distributions
- Running long jobs and scripts in tmux and checking back on them later
- Learning Arch and pacman without installing it on your own hardware

## Dependencies for Arch Linux Web Terminal Hosting

- A Railway volume mounted at `/home` (included in the template)
- Nothing else: no database, no API keys, no external accounts

### Deployment Dependencies

- Arch Linux: https://archlinux.org/
- Official Arch Linux image on Docker Hub: https://hub.docker.com/_/archlinux
- Arch User Repository (AUR): https://wiki.archlinux.org/title/Arch_User_Repository
- ttyd (web terminal, MIT): https://github.com/tsl0922/ttyd
- tmux: https://github.com/tmux/tmux/wiki
- Railway build variables: https://docs.railway.com/builds/dockerfiles#using-variables-at-build-time
- Railway Fair Use Policy: https://railway.com/legal/fair-use

### Implementation Details

| Service | Source | Purpose |
|---|---|---|
| Terminal | Dockerfile based on `archlinux:base-20261108.0.623074` | nginx basic-auth proxy on port 8080 (public HTTPS domain, `/healthz`), ttyd on loopback, tmux sessions, user `arch` with sudo, volume at `/home` |

**Plan:** works on every plan, including Free (0.5 GB RAM per service). Measured on Railway with a 0.5 GB memory limit: about 0.035 GB of RAM idle, and about 0.18 GB (page cache included) right after an `npm install -g`; an AUR build of `yay-bin` with `makepkg` also ran fine. The image is about 0.45 GB compressed (single layer). Large builds can need more than 0.5 GB; on Free the kernel then stops that process, not the terminal.

**First login**

1. Open the service's public URL.
2. Your browser asks for a user name and password. The user name is `arch` (the `USERNAME` variable). The password is the generated `PASSWORD` value on the service's Variables tab.
3. You land in a tmux session named `main`. Close the tab at any time and reopen the URL to resume. For a second, independent session, open `https://{your-domain}/?arg=work`.
4. `sudo -i` gives you a root shell.

**Tools included**: base-devel (gcc, make, makepkg, sudo and friends), git, git-lfs, curl, wget, vim, nano, tmux, htop, ncdu, tree, jq, ripgrep, fd, rsync, sqlite3, zip/unzip, dig, ping, netcat, openssh, Python 3 with pip and pipx, and Node.js 24 LTS with npm. `npm install -g` installs into `~/.npm-global`, so it works without sudo and persists. Arch's Python is marked as externally managed: use `python -m venv` or `pipx` for packages.

**AUR**: you are a normal user with sudo, which is what `makepkg` expects. For example: `git clone https://aur.archlinux.org/yay-bin.git && cd yay-bin && makepkg -si`. AUR packages are user-produced content; read each PKGBUILD before you build it.

**More packages**: set the `ARCH_EXTRA_PACKAGES` variable to a space-separated list of packages or groups from the official repositories, for example `go rust`. Railway passes the variable to the Docker build, so the next deploy builds an image with those packages and they survive every redeploy. Names pacman cannot install are skipped with a warning in the build and deploy logs, so a typo never fails the deploy. Packages you install with `sudo pacman -Syu PACKAGE` in the shell work right away but reset on the next redeploy. The image keeps no man pages (Arch's container image excludes them).

**What persists**: everything under `/home/arch`: files, Git repositories, AUR build directories, shell history, dotfiles, Python virtualenvs and pipx apps in `~/.local`, global npm packages in `~/.npm-global`. Running processes and packages installed with pacman reset on every redeploy or restart. tmux keeps sessions alive across browser reconnects, not across container restarts.

**Security**: the shell is never served without a password. With an empty, short (under 12 characters) or overlong `PASSWORD` the service still starts, but the login is locked with a random password nobody knows and the deploy log says how to fix it. Invalid `PORT`, `USERNAME` or `TERMINAL_FONT_SIZE` values fall back to the defaults with a warning in the log. nginx checks the password, which the container keeps only as a SHA-512 crypt hash, and rate-limits attempts. It also rejects WebSocket connections from other sites. ttyd listens on 127.0.0.1 only. The password does not appear in logs, process arguments or the shell environment. To change it, edit `PASSWORD` and redeploy. Browsers cache basic-auth logins until you close them, so use a private window on shared computers.

**Limits**: this is a container, not a VM. There is no systemd, no Docker inside the shell, no raw sockets (`ping` works with `sudo`), and only HTTP(S) on the public domain. Railway's [Fair Use Policy](https://railway.com/legal/fair-use) applies to everything that runs in this shell.

**Scaling**: this is a single-user machine on one volume, so keep one replica. Railway scales RAM and CPU vertically up to your plan limit. For a team, deploy one terminal per person.

**Pinning and upgrades**: Arch is a rolling release. The base image is pinned by dated tag and digest on the `FROM` line of `services/terminal/Dockerfile`, and every build runs `pacman -Syu`, so a redeploy that rebuilds the image brings the whole system up to date. A pacman upgrade inside the shell lasts until the next redeploy.

### Why Deploy Arch Linux Web Terminal on Railway?

Railway gives the terminal an HTTPS URL, a persistent volume and automatic restarts, with no server to administer. You get a protected, always-current Arch Linux shell from one click and pay only for the CPU and memory it uses. This is a community template, not affiliated with Arch Linux, the ttyd author or Railway. Arch Linux is a trademark of the Arch Linux project.
