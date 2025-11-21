# pyenv for Termux: build + deploy plan

Goal: install pyenv on the target Termux device and build/use CPython versions safely. The build box (milkneko-dev) is available if we want to offload or cache downloads.

## Hosts
- Build box: `milkneko-dev` (Ubuntu; NDK r26d already present under /opt/android-ndk).
- Target box: `luis-tab-s8` (Termux on Android; aarch64/Bionic; reachable via SSH).

## Project specifics (svc-core)
- Repo paths: target `/data/data/com.termux/files/home/lyucra/svc-core`; reference workstation copy `/Volumes/ExternalDisk/LUIS_DATA/WORLD_WINNER/svc-core`.
- `.python-version` expects pyenv env `app-3.11` → on workstation this maps to CPython **3.11.14** (`pyenv versions` shows `app-3.11 -> 3.11.14/envs/app-3.11`).
- Target currently **has no pyenv installed** (`pyenv versions` fails).
- Packages that need native deps: `cryptography` (openssl, rust), `psycopg` (libpq/pg_config), `pillow` (jpeg/png/freetype/lcms), `cffi` (libffi), `brotli` (C build). Plan Termux pkg install accordingly.
- Disk on target: ~49 GB free (`df -h ~`); kernel `5.10.236-android` aarch64; should tolerate native builds with modest parallelism (e.g., `MAKEFLAGS=-j4`).

## Inputs to confirm
- Python versions needed (e.g., 3.11.14, 3.12.x).
- Disk budget on the tablet and acceptable build parallelism (e.g., `MAKEFLAGS=-j4`).
- Whether cross-compiling on the build box is mandatory; pyenv defaults to native builds and cross-compiling for Android is fragile.

## Decision guidance
- Recommended: build CPython natively inside Termux with pyenv; the Termux toolchain already targets Bionic correctly.
- Only if offloading is mandatory: attempt cross-compiling with NDK on the build box and rsync the resulting pyenv versions; this needs extra patches and is not the default path.

## Plan (happy path — native Termux build)
1) **Inspect target**
   ```bash
   ssh luis-tab-s8 'uname -a; df -h ~'
   ssh luis-tab-s8 'pyenv --version || true'
   ```
2) **Install build deps on Termux** (adjust if any package names differ):
   ```bash
   ssh luis-tab-s8 'pkg update -y && pkg install -y \
     make clang git pkg-config python \
     rust \
     openssl openssl-tool \
     libffi zlib bzip2 \
     readline sqlite liblzma libcrypt libuuid \
     postgresql \
     libjpeg-turbo libpng freetype libtiff libwebp littlecms'
   ```
3) **Build CPython with pyenv (Android/Bionic quirks)**
   - Bionic misses several glibc calls; pass these to `pyenv install`:
     ```bash
     export PYTHON_ANDROID_MISSING_FUNCS="
       ac_cv_func_sem_clockwait=no
       ac_cv_func_close_range=no
       ac_cv_func_copy_file_range=no
       ac_cv_func_getloadavg=no
       ac_cv_func_pwritev2=no
       ac_cv_func_preadv2=no
       ac_cv_func_fexecve=no
       ac_cv_func_getpwent=no
       ac_cv_func_getlogin_r=no
       ac_cv_func_pthread_getname_np=no
       ac_cv_header_spawn_h=no
       ac_cv_func_posix_spawn=no
       ac_cv_func_posix_spawnp=no
     "
     ```
   - Use Termux headers/libs explicitly and avoid shared lib hardlinking:
     ```bash
     export CPPFLAGS="-I$PREFIX/include -I$PREFIX/include/ncursesw"
     export LDFLAGS="-L$PREFIX/lib"
     export PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig"
     export PYTHON_CONFIGURE_OPTS="--disable-shared --with-system-ffi"
     export MAKEFLAGS=-j4
     ```
   - Example install commands:
     ```bash
     for v in 3.11.14 3.12.12 3.13.9 3.14.0; do
       ssh luis-tab-s8 "bash -lc 'source ~/.bashrc && \
         $PYTHON_ANDROID_MISSING_FUNCS \
         CPPFLAGS=\"$CPPFLAGS\" LDFLAGS=\"$LDFLAGS\" PKG_CONFIG_PATH=\"$PKG_CONFIG_PATH\" \
         PYTHON_CONFIGURE_OPTS=\"$PYTHON_CONFIGURE_OPTS\" MAKEFLAGS=$MAKEFLAGS \
         pyenv install $v'"
     done
     ```
   - Create the project virtualenv:
     ```bash
     ssh luis-tab-s8 "bash -lc 'source ~/.bashrc && pyenv virtualenv 3.11.14 app-3.11'"
     ```
3) **Install pyenv on Termux**
   ```bash
   ssh luis-tab-s8 'git clone https://github.com/pyenv/pyenv.git ~/.pyenv'
   ssh luis-tab-s8 "printf '\nexport PYENV_ROOT=\"$HOME/.pyenv\"\nexport PATH=\"$PYENV_ROOT/bin:$PATH\"\nif command -v pyenv >/dev/null 2>&1; then eval \"$(pyenv init -)\"; fi\n' >> ~/.bashrc"
   # for zsh shells, append the same to ~/.zshrc
   ```
   Reload shell or `ssh luis-tab-s8 'source ~/.bashrc && pyenv --version'` to confirm.
4) **Build Python versions on Termux**
   ```bash
   # example for Python 3.11.14
   ssh luis-tab-s8 'source ~/.bashrc && \
     MAKEFLAGS=-j4 PYTHON_CONFIGURE_OPTS="--enable-shared" \
     pyenv install 3.11.14'
   ssh luis-tab-s8 'source ~/.bashrc && pyenv global 3.11.14 && python -V'
   ssh luis-tab-s8 'source ~/.bashrc && python -c "import ssl, sqlite3; print(ssl.OPENSSL_VERSION)"'
   ```
5) **Cache sources (optional)**
   ```bash
   ssh milkneko-dev 'mkdir -p /opt/milkneko/pyenv_cache'
   # symlink pyenv download cache if you want to reuse tarballs between hosts
   ssh luis-tab-s8 'mkdir -p ~/.pyenv/cache'
   # rsync tarballs from build box when needed
   ```

## Alternate path (only if cross-compile is required)
- Use NDK clang triple `aarch64-linux-android33-` on the build box and patch python-build to accept Android; set `CC`, `CXX`, `AR`, and `RANLIB` env vars for `pyenv install`.
- Tar the resulting `~/.pyenv/versions/<ver>` and `~/.pyenv/versions/<ver>/lib/python*/config-*/Makefile` onto the target; ensure `_sysconfigdata__` matches Android.
- Expect manual fixes for `_ctypes`, `multiprocessing/semaphore` and OpenSSL paths; this route is experimental and slower than native Termux builds.

## Validation checklist
- `pyenv versions` shows the installed interpreters on Termux.
- `python -V` matches the requested version.
- Basic stdlib sanity: `python - <<'PY'\nimport ssl, sqlite3, sys\nprint('ssl ok', ssl.OPENSSL_VERSION)\nprint('sqlite ok', sqlite3.sqlite_version)\nprint('prefix', sys.prefix)\nPY`
- If cross-built, run a small script that touches `multiprocessing`, `ssl`, and `subprocess` to confirm no missing syscalls.

## Current status (Nov 21, 2025)
- Termux pyenv installed; shell init blocks added to `~/.bashrc` and `~/.zshrc`.
- Installed interpreters: 3.11.14 (plus `app-3.11` venv), 3.12.12, 3.13.9, 3.14.0.
- `app-3.11` is ready for `svc-core` (.python-version already points to it).
- Sanity check for 3.11.14 env: `ssl`, `ctypes`, and `readline` extensions load successfully.

## Next actions
- Confirm desired Python versions and whether cross-compiling is truly required.
- Run the Termux dependency install and pyenv bootstrap commands (steps 1–4).
- Add any host-specific tweaks to this file as you execute them.
