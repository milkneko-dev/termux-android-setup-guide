# Building OpenAI Codex CLI for Termux (Android)

This document captures everything we did to make the Codex CLI work directly in
Termux without relying on the Ubuntu proot container. Keep it around so future
updates to Codex can follow the same recipe.

---

## 1. Background & Symptoms

- Official Codex binaries are statically linked (musl). On this device Cloudflare
  silently drops the TLS handshake for this fingerprint, so every
  `https://chatgpt.com/backend-api/codex/responses` call fails with
  `error sending request` after 5 retries.
- Newer Codex versions also call `prctl(PR_SET_DUMPABLE, 0)` before any CLI flags
  are processed. That breaks inside proot unless the binary tolerates the prctl
  failure.
- Solution: build our own Android/Bionic binary so it runs **inside Termux** and
  therefore presents a normal Android TLS handshake (Cloudflare stops rejecting
  it). We also patched the musl build to optionally skip the PRCTL error, just in
  case we ever need to run inside proot again.

---

## 2. Build Host Prerequisites (`milkneko-dev`)

```bash
sudo apt-get update
sudo apt-get install -y build-essential pkg-config libssl-dev clang cmake unzip curl git
```

Install Rust via rustup (default answers):

```bash
cd /opt/milkneko
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env
```

Add the Rust targets we need:

```bash
rustup target add aarch64-unknown-linux-gnu
rustup target add aarch64-linux-android
# Codex pins to 1.90.0 in rust-toolchain, so install the targets for that toolchain too
rustup target add --toolchain 1.90.0-x86_64-unknown-linux-gnu aarch64-unknown-linux-gnu aarch64-linux-android
```

Install the Android NDK (we used r26d) under `/opt/android-ndk`:

```bash
cd /opt
sudo mkdir -p android-ndk && sudo chown $USER android-ndk
cd android-ndk
wget https://dl.google.com/android/repository/android-ndk-r26d-linux.zip
unzip android-ndk-r26d-linux.zip
rm android-ndk-r26d-linux.zip
export NDK=/opt/android-ndk/android-ndk-r26d
```

---

## 3. Clone Codex and apply local patches

```bash
cd /opt/milkneko
rm -rf codex-src
git clone https://github.com/openai/codex codex-src
cd codex-src
```

### 3.1 Allow PR_SET_DUMPABLE to fail (optional musl fix)

Edit `codex-rs/process-hardening/src/lib.rs` so it tolerates the PRCTL failure
when the env var `CODEX_ALLOW_PRCTL_FAILURE=1` is present.

```rust
let ret_code = unsafe { libc::prctl(libc::PR_SET_DUMPABLE, 0, 0, 0, 0) };
if ret_code != 0 {
    let err = std::io::Error::last_os_error();
    if std::env::var_os("CODEX_ALLOW_PRCTL_FAILURE").is_some() {
        eprintln!("WARNING: prctl(PR_SET_DUMPABLE, 0) failed ({}); continuing because CODEX_ALLOW_PRCTL_FAILURE is set.", err);
    } else {
        eprintln!("ERROR: prctl(PR_SET_DUMPABLE, 0) failed: {err}");
        std::process::exit(PRCTL_FAILED_EXIT_CODE);
    }
}
```

### 3.2 Teach the workspace about Android OpenSSL

Append the following block to `codex-rs/core/Cargo.toml` so the vendored OpenSSL
build is used for Android just like it already is for musl:

```toml
[target.aarch64-linux-android.dependencies]
openssl-sys = { workspace = true, features = ["vendored"] }
```

---

## 4. Build outputs

### 4.1 Termux-native binary (Bionic/glibc)

> **Version note (Nov 2025):** the current upstream release is **0.56.0**. Make
> sure `[workspace.package] version` in `codex-rs/Cargo.toml` matches whatever
> tag you are targeting before you run the commands below.

```bash
export NDK=/opt/android-ndk/android-ndk-r26d
cd /opt/milkneko/codex-src/codex-rs
source ~/.cargo/env
CC_aarch64_linux_android=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android33-clang \
CXX_aarch64_linux_android=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android33-clang++ \
AR_aarch64_linux_android=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar \
CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android33-clang \
PKG_CONFIG_ALLOW_CROSS=1 \
cargo build --release --target aarch64-linux-android --bin codex
```

Result: `target/aarch64-linux-android/release/codex` (~37 MB). This binary runs
inside Termux without proot and survives device restarts.

### 4.2 Optional MUSL/gnu build for Ubuntu proot

```bash
source ~/.cargo/env
CC_aarch64_unknown_linux_gnu=aarch64-linux-gnu-gcc \
CXX_aarch64_unknown_linux_gnu=aarch64-linux-gnu-g++ \
AR_aarch64_unknown_linux_gnu=aarch64-linux-gnu-ar \
PKG_CONFIG_ALLOW_CROSS=1 \
CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc \
cargo build --release --target aarch64-unknown-linux-gnu --bin codex
```

If you run that inside proot, export `CODEX_ALLOW_PRCTL_FAILURE=1` before launching.

---

## 5. Deploy to the phone (Termux side)

From the macOS host:

```bash
# Copy the Android binary into Termux
scp /tmp/codex-android luis-tab-s8:/data/data/com.termux/files/home/codex-android

# Make it executable and wire it into the PATH
dash
ssh luis-tab-s8 'chmod +x /data/data/com.termux/files/home/codex-android'
ssh luis-tab-s8 'ln -sfn /data/data/com.termux/files/home/codex-android /data/data/com.termux/files/usr/bin/codex'
```

Test after a Termux restart:

```bash
adb shell am force-stop com.termux
adb shell am start -n com.termux/.app.TermuxActivity
ssh luis-tab-s8 'codex exec --skip-git-repo-check -- "Hello from Termux"'
```

The command should complete immediately with a valid response. No environment
variables are required because the Android binary’s TLS fingerprint matches what
Cloudflare expects from this device.

### 5.1 Optional: update Ubuntu proot distro

Termux and proot Ubuntu do not share `/usr/bin` or `/usr/local/bin`. Updating
the Termux symlink above does not update the Ubuntu rootfs. If you use proot,
deploy the GNU build from section 4.2 as well:

```bash
# Copy the GNU/proot binary into Termux home
scp /tmp/codex-aarch64-gnu luis-tab-s8:/data/data/com.termux/files/home/codex-aarch64-gnu

# Install it inside the Ubuntu rootfs
ssh luis-tab-s8 'proot-distro login ubuntu -- sh -lc "install -m 755 /data/data/com.termux/files/home/codex-aarch64-gnu /usr/local/bin/codex"'
```

Because proot blocks `prctl(PR_SET_DUMPABLE, 0)`, set the override before
launching:

```bash
ssh luis-tab-s8 'proot-distro login ubuntu -- env CODEX_ALLOW_PRCTL_FAILURE=1 codex --version'
```

If you want that env var to be automatic, add it to the Ubuntu shell profile
(for example `/root/.bashrc` when using the default root login).

---

## 6. Maintenance tips

- Keep `/opt/milkneko/codex-src` around with the two local patches applied. To
  pick up new upstream releases: `git pull`, re-apply the patch if it gets
  overwritten, and re-run the build commands above.
- When bumping Codex to a new release (e.g. 0.56.0):
  1. `cd /opt/milkneko/codex-src && git pull`.
  2. Update `[workspace.package] version` inside `codex-rs/Cargo.toml` to match the GitHub release tag.
  3. Re-run the Android (and optional musl) build commands from sections 4.1/4.2.
  4. Copy the new binary onto the phone, keep the `/data/data/com.termux/files/usr/bin/codex` symlink,
     and verify `codex --version` reports the fresh version.
  5. If you use proot Ubuntu, also deploy the GNU build via section 5.1 and verify
     inside proot with `CODEX_ALLOW_PRCTL_FAILURE=1 codex --version`.

- If Cloudflare ever reverts its filtering, you can swap the symlink back to the
  stock npm binary, but the Android build is safer because it doesn’t depend on
  proot or `CODEX_ALLOW_PRCTL_FAILURE`.
- For proot testing, the MUSL build plus
  `CODEX_ALLOW_PRCTL_FAILURE=1 /path/to/codex-aarch64-gnu …` still works.
- The git helper for this repo uses `gh auth git-credential`, so log into GitHub
  with `gh auth login` whenever you move to a new machine.

With these notes we can always rebuild and ship a Termux-friendly Codex CLI in a
few minutes instead of reverse-engineering the whole process again.
