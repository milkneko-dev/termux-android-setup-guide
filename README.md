# Termux Android Setup Guide

Personal notes for provisioning Termux on a Samsung tablet (Android 13/One UI 6) from macOS, smoothing UI distractions, enabling clipboard sharing, and installing Nerd Fonts for remote dev work.

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

Example macOS alias for sending your current clipboard into Termux:

```bash
alias termuxclip="pbpaste | adb shell 'run-as com.termux /data/data/com.termux/files/usr/bin/bash -lc \"PATH=$PREFIX/bin:$PREFIX/bin/applets:\$PATH; LD_LIBRARY_PATH=$PREFIX/lib:\$LD_LIBRARY_PATH; export PATH LD_LIBRARY_PATH; termux-clipboard-set\"'"
```

Grant the clipboard permission prompt the first time the companion app requests it.

## 7. Install Nerd Fonts inside Termux
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

## 8. Handy maintenance commands
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

## 9. Troubleshooting notes
- If `pm install` cannot read from `/sdcard/Download/…`, push APKs to `/data/local/tmp` first.
- Clipboard errors like `No shell command implementation.` mean the Samsung build blocks ADB clipboard APIs; Termux:API is the workaround.
- When using `run-as com.termux`, always prepend `PATH=$PREFIX/bin:$PREFIX/bin/applets:$PATH` so Termux binaries resolve.

Document last reviewed: **2025-09-29**.
