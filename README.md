# Termux Android Setup Guide

Personal notes for provisioning Termux on a Samsung tablet (Android 15/One UI 7) from macOS, smoothing UI distractions, enabling clipboard sharing, and installing Nerd Fonts for remote dev work.

## 1. Prerequisites
- Android device (Samsung tablet/phone) with developer options enabled and USB debugging allowed.
- macOS workstation with `adb` (Android platform-tools) and `gh` installed.
- USB cable for the device.
- Optionally, `pbpaste`/`pbcopy` integration on macOS.

Verify connection:

```bash
adb devices
```

## 2. Install Termux from GitHub
1. Enable *Settings → Security & privacy → Install unknown apps* for the browser or file manager you will use.
2. Download the latest Termux APK directly from the official GitHub releases. For this device the ABI reported by `adb shell getprop ro.product.cpu.abi` is `arm64-v8a`, so grab:

   ```
   https://github.com/termux/termux-app/releases/download/v0.118.3/termux-app_v0.118.3+github-debug_arm64-v8a.apk
   ```

3. Push and install via `adb` (replace the path to match your download location):

   ```bash
   adb push ~/Downloads/termux-app_v0.118.3+github-debug_arm64-v8a.apk /data/local/tmp/termux.apk
   adb shell pm install -r /data/local/tmp/termux.apk
   ```

4. Launch Termux once from the app drawer to complete initialization.

## 3. Initial Termux bootstrap
Update packages and install helpers:

```bash
pkg update -y && pkg upgrade -y
pkg install -y git openssh wget unzip
```

If you ever run package commands via `adb shell run-as`, export the Termux paths first:

```bash
run-as com.termux /data/data/com.termux/files/usr/bin/bash -lc \
  'PATH=$PREFIX/bin:$PREFIX/bin/applets:$PATH; \
   LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH; \
   export PATH LD_LIBRARY_PATH; pkg update'
```

## 4. Hide Samsung taskbar and navigation buttons
To make Termux fullscreen:

1. **Taskbar** – Settings → Display → *Taskbar* → toggle **Off** (or enable *Hide taskbar when using apps*).
2. **Navigation bar** – Settings → Display → *Navigation bar* → choose **Swipe gestures** so the on-screen buttons disappear.
3. If the S Pen icon remains, disable it under Settings → Advanced features → S Pen → toggle off Air Command shortcuts.

## 5. Termux fullscreen and UI tweaks
Create or edit `~/.termux/termux.properties` inside Termux:

```bash
mkdir -p ~/.termux
cat <<'EOP' > ~/.termux/termux.properties
fullscreen = true
use-fullscreen-workaround = true
extra-keys = []
EOP
termux-reload-settings
```

- `fullscreen` hides the Android status bar.
- `use-fullscreen-workaround` keeps the keyboard visible on some Samsung builds.
- `extra-keys = []` removes Termux’s bottom key row (toggle back temporarily with `Volume Up + K`).

## 6. Clipboard integration (Termux:API)
Install the termux-api package and companion app:

```bash
pkg install -y termux-api
```

Download and install the matching APK (latest release as of 2025-09-29 is v0.53.0):

```bash
curl -L -o ~/Downloads/termux-api-app_v0.53.0+github.debug.apk \
  https://github.com/termux/termux-api/releases/download/v0.53.0/termux-api-app_v0.53.0%2Bgithub.debug.apk
adb push ~/Downloads/termux-api-app_v0.53.0+github.debug.apk /data/local/tmp/termux-api.apk
adb shell pm install -r /data/local/tmp/termux-api.apk
```

Now `termux-clipboard-set` and `termux-clipboard-get` work. To seed the clipboard from macOS:

```bash
adb shell "run-as com.termux /data/data/com.termux/files/usr/bin/bash -lc \
  'PATH=$PREFIX/bin:$PREFIX/bin/applets:$PATH; \
   LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH; \
   export PATH LD_LIBRARY_PATH; termux-clipboard-set "'\''text here'\''"'"
```

Inside Termux itself, pipe anything you want into the Android clipboard or read it back again:

```bash
echo "text here" | termux-clipboard-set     # copy literal text
cat notes.txt | termux-clipboard-set        # send a file's contents
termux-clipboard-get > pasted.txt           # capture clipboard into a file
```

Add macOS aliases for sending your clipboard into Termux and pulling the Termux clipboard back to macOS:

```bash
# ~/.zshrc (or similar)
alias termuxclippush='pbpaste | adb shell '\''run-as com.termux /data/data/com.termux/files/usr/bin/bash -lc "PATH=$PREFIX/bin:$PREFIX/bin/applets:\$PATH; LD_LIBRARY_PATH=$PREFIX/lib:\$LD_LIBRARY_PATH; export PATH LD_LIBRARY_PATH; termux-clipboard-set"'\''
alias termuxclippull='adb shell '\''run-as com.termux /data/data/com.termux/files/usr/bin/bash -lc "PATH=$PREFIX/bin:$PREFIX/bin/applets:\$PATH; LD_LIBRARY_PATH=$PREFIX/lib:\$LD_LIBRARY_PATH; export PATH LD_LIBRARY_PATH; termux-clipboard-get"'\'' | pbcopy'
```

Usage:

- `termuxclippush` — copies the current macOS clipboard into the Android clipboard.
- `termuxclippull` — replaces the macOS clipboard with whatever is currently selected in Termux/Android.

Grant the clipboard permission prompt the first time the companion app requests it.

### Termux API inside proot-distro Ubuntu
Android 14+ blocks `app_process` execs from inside proot, so route Termux:API calls through the socket client instead.

1. **Prep the host Termux session (outside proot):**
   ```bash
   pkg install termux-api termux-am-socket
   ```
   Open the Termux:API Android app once, grant Clipboard permission, then confirm the socket server is alive:
   ```bash
   echo "$TERMUX_APP__AM_SOCKET_SERVER_ENABLED"
   ```
   Expect `true`; if empty, force-stop Termux and relaunch.

2. **Expose Termux binaries in Ubuntu without overriding shims:** append the Termux prefix to PATH (and optionally LD libs) from inside the Ubuntu rootfs:
   ```bash
   sudo tee /etc/profile.d/termux-host.sh >/dev/null <<'EOF'
   #!/bin/sh
   TERMUX_PREFIX="/data/data/com.termux/files/usr"
   if [ -d "$TERMUX_PREFIX/bin" ]; then
     PATH="$PATH:$TERMUX_PREFIX/bin"
     export PATH
   fi
   if [ -d "$TERMUX_PREFIX/lib" ]; then
     LD_LIBRARY_PATH="${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}$TERMUX_PREFIX/lib"
     export LD_LIBRARY_PATH
   fi
   EOF
   ```

3. **Shadow `am` with the socket-aware client:**
   ```bash
   sudo tee /usr/local/bin/am >/dev/null <<'EOF'
   #!/bin/sh
   exec /data/data/com.termux/files/usr/bin/termux-am "$@"
   EOF
   sudo chmod +x /usr/local/bin/am
   ```

4. **Verify from a fresh proot shell:**
   ```bash
   which am
   which termux-clipboard-get
   termux-clipboard-set "test from proot"
   termux-clipboard-get
   ```
   Expect `/usr/local/bin/am` and clipboard round-trips without the `app_process` error.

5. **Fallback (only if the socket path fails):** before launching proot, copy a working `app_process64` into Termux and bind it in your alias, e.g.:
   ```bash
   install -m 0755 /apex/com.android.art/bin/app_process64 \
     $PREFIX/libexec/app_process.termux
   proot-distro login ubuntu --bind \
     $PREFIX/libexec/app_process.termux:/system/bin/app_process
   ```
   This restores the legacy `am` behavior but is slower than the socket workaround.

## 7. SSH host autocompletion in Termux
Install bash-completion so `ssh` picks up host aliases from `~/.ssh/config` and `~/.ssh/known_hosts`:

```bash
pkg install -y bash-completion
```

If you do not already have a `~/.bashrc`, create one and source the Termux bash-completion shim:

```bash
cat <<'EOF' >> ~/.bashrc
# Enable bash completion (SSH hosts, git, etc.)
if [ -f "$PREFIX/etc/profile.d/bash_completion.sh" ]; then
  . "$PREFIX/etc/profile.d/bash_completion.sh"
fi
EOF
```

For login shells (Termux launches bash as a login shell), make sure `~/.bash_profile` sources `~/.bashrc`:

```bash
cat <<'EOF' >> ~/.bash_profile
if [ -f "$HOME/.bashrc" ]; then
  . "$HOME/.bashrc"
fi
EOF
```

Restart Termux or run `source ~/.bashrc` in the current session. Tab-completion now expands aliases like:

```bash
ssh ww<Tab>
```

## 8. Install Nerd Fonts inside Termux
Use Nerd Fonts so remote Neovim icons render correctly:

```bash
mkdir -p ~/fonts && cd ~/fonts
wget https://github.com/ryanoasis/nerd-fonts/releases/download/v3.1.1/FiraCode.zip
unzip -o FiraCode.zip
mkdir -p ~/.termux
cp FiraCodeNerdFontMono-Regular.ttf ~/.termux/font.ttf
```

Restart Termux after changing `font.ttf`:

```bash
adb shell am force-stop com.termux
adb shell am start -n com.termux/.app.TermuxActivity
```

Swap in another style (e.g., `FiraCodeNerdFont-SemiBold.ttf`) whenever needed and restart again.

## 9. Handy maintenance commands
- Restart Termux quickly:
  ```bash
  adb shell am force-stop com.termux && adb shell am start -n com.termux/.app.TermuxActivity
  ```
- Check kernel version:
  ```bash
  adb shell uname -a
  ```
- Install OpenSSH client inside Termux:
  ```bash
  pkg install openssh
  ssh-keygen -t ed25519 -C "device-termux"
  ```

## 10. Troubleshooting notes
- If `pm install` cannot read from `/sdcard/Download/…`, push APKs to `/data/local/tmp` first.
- Clipboard errors like `No shell command implementation.` mean the Samsung build blocks ADB clipboard APIs; Termux:API is the workaround.
- When using `run-as com.termux`, always prepend `PATH=$PREFIX/bin:$PREFIX/bin/applets:$PATH` so Termux binaries resolve.

## 11. Install Neovim with NvChad (Termux)
1. Fetch the latest stable Neovim tarball and drop it under `~/.local` (Termux packages require sudo for system-wide installs):

   ```bash
   mkdir -p ~/.local ~/.config
   curl -L --fail -o ~/nvim-linux-arm64.tar.gz \
     https://github.com/neovim/neovim/releases/download/stable/nvim-linux-arm64.tar.gz
   tar -xzf ~/nvim-linux-arm64.tar.gz -C ~/.local
   rm ~/nvim-linux-arm64.tar.gz
   ```

2. Add the Neovim bin directory to your PATH so `nvim` resolves in future sessions:

   ```bash
   cat <<'EOF' >> ~/.bashrc
   if [ -d "$HOME/.local/nvim-linux-arm64/bin" ]; then
     case ":$PATH:" in
       *:"$HOME/.local/nvim-linux-arm64/bin":*) ;;
       *) PATH="$HOME/.local/nvim-linux-arm64/bin:$PATH" ;;
     esac
   fi
   EOF
   source ~/.bashrc
   ```

   For login shells, append the same guard to `~/.profile` so Termux picks it up on launch:

   ```bash
   cat <<'EOF' >> ~/.profile
   if [ -d "$HOME/.local/nvim-linux-arm64/bin" ] ; then
     PATH="$HOME/.local/nvim-linux-arm64/bin:$PATH"
   fi
   EOF
   ```

3. Install the NvChad starter config and sync plugins:

   ```bash
   rm -rf ~/.config/nvim
   git clone https://github.com/NvChad/starter ~/.config/nvim --depth 1
   ~/.local/nvim-linux-arm64/bin/nvim --headless "+Lazy! sync" +qa
   ```

4. Launch Neovim normally (`nvim`). NvChad will finish lazy-loading plugins on first run. Use `Ctrl+n` to toggle the file tree and `<space> e` to focus it.

## 12. PostgreSQL for `svc-core` tests
The `svc-core` repository expects a local PostgreSQL instance during its test suite. On the Ubuntu host:

```bash
sudo apt-get update
sudo apt-get install -y postgresql
```

If the package post-install script cannot initialize `/var/lib/postgresql/17/main` (common on user-namespaced environments), create a user-owned data directory instead:

```bash
/usr/lib/postgresql/17/bin/initdb -D ~/postgres-data \
  --auth-local=trust --auth-host=scram-sha-256
/usr/lib/postgresql/17/bin/pg_ctl -D ~/postgres-data -l ~/postgres-data/logfile start
/usr/lib/postgresql/17/bin/psql -d postgres -c 'SELECT 1;'
```

Optional: create dedicated databases for development and testing.

```bash
/usr/lib/postgresql/17/bin/createdb svc_core_dev
/usr/lib/postgresql/17/bin/createdb svc_core_test
```

When finished, stop the instance cleanly:

```bash
/usr/lib/postgresql/17/bin/pg_ctl -D ~/postgres-data stop
```

Export `DATABASE_URL=postgresql://localhost:5432/postgres` (or one of the new databases) before running `svc-core` tests.

If you rely on the decrypted `.env`, create the matching PostgreSQL superuser:

```bash
/usr/lib/postgresql/17/bin/psql -d postgres -c "CREATE ROLE root WITH SUPERUSER LOGIN PASSWORD 'root';"
```

With the virtualenv active (`source .venv/bin/activate`), `make reset-database` now boots a clean schema against the local instance (expect harmless warnings about optional psycopg binaries).

If PostgreSQL 17 triggers platform-specific regressions, install PostgreSQL 14 from PGDG instead and run it on an alternate port (5433 shown below):

```bash
# add PGDG repository (only once)
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/pgdg.asc] http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main' |   sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt-get update

# install the 14.x binaries
sudo apt-get install -y libicu70 libldap-2.5-0 postgresql-14 postgresql-client-14

# initialize a user-scoped data directory and start on port 5433
/usr/lib/postgresql/14/bin/initdb -D ~/postgres14-data --auth-local=trust --auth-host=scram-sha-256
/usr/lib/postgresql/14/bin/pg_ctl -D ~/postgres14-data -o "-p 5433" -l ~/postgres14-data/logfile start
/usr/lib/postgresql/14/bin/psql -p 5433 -d postgres -c "CREATE ROLE root WITH SUPERUSER LOGIN PASSWORD 'root';"
```

Update the project `.env` to point at `psql://root:root@localhost:5433/project-x` and export `PGHOST=localhost PGPORT=5433 PGUSER=root PGPASSWORD=root` when running `make reset-database` or `make test`.



Document last reviewed: **2025-10-08**.

## 13. svc-core virtualenv & secrets
Before running Django management commands:

```bash
# ensure uv is available for Python 3.11 builds
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.11
uv venv --python 3.11
source .venv/bin/activate
pip install -r requirements/dev.txt
```

After syncing Python deps, install GNU gettext (needed for message catalogs) and compile translations so CTA labels render like desktop builds:

```bash
sudo apt-get install -y gettext
source .venv/bin/activate
python manage.py compilemessages
```
The secrets fixtures rely on the `ejson` CLI:

```bash
sudo apt-get install -y golang-go
go install github.com/Shopify/ejson/cmd/ejson@latest
sudo ln -s "$HOME/go/bin/ejson" /usr/local/bin/ejson
```

Then decrypt the local environment secrets:

```bash
source .venv/bin/activate
make secrets-create
```

The command writes `.env` at the repository root so `SECRET_KEY` and other settings resolve for `manage.py` invocations (e.g., `make reset-database`).

## 14. Redis for `svc-core`
Install the bundled Redis server on Ubuntu and start the daemon:

```bash
sudo apt-get install -y redis-server
sudo service redis-server start
```

The init script may log an `ulimit` warning under user namespaces; the server still launches. Verify the instance responds locally:

```bash
redis-cli ping
# PONG
```

If you need Redis automatically for future sessions, keep the `redis-server` service running or add it to your startup scripts before running `make` targets that depend on it.

## 15. AWS CLI (Ubuntu host)
Install AWS CLI v2 inside the proot Ubuntu environment without requiring `sudo`:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o /tmp/awscliv2.zip
rm -rf /tmp/aws
unzip -q /tmp/awscliv2.zip -d /tmp
/tmp/aws/install -i "$HOME/.local/aws-cli" -b "$HOME/.local/bin"
```

Verify the binary resolves (Termux shells already export `~/.local/bin` onto `PATH`):

```bash
aws --version
# aws-cli/2.31.10 Python/3.13.7 ...
```

If `aws` is not found, add `~/.local/bin` to your PATH in `~/.bashrc`:

```bash
case ":$PATH:" in
  *:"$HOME/.local/bin":*) ;;
  *) PATH="$HOME/.local/bin:$PATH" ;;
esac
export PATH
```
