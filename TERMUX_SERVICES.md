# Termux Service Management

Notes on using the `termux-services` package and runit to supervise daemons (e.g., `sshd`) inside this Termux setup.

## Install prerequisites
- Open Termux and update packages: `pkg update && pkg upgrade`.
- If `pkg install` refuses to fetch from `mirror.albony.in`, run `termux-change-repo` and select *Packages (packages-cf.termux.dev)*.
- Install the supervisor bundle (pulls in runit):
  ```bash
  pkg install termux-services
  ```
- (Optional) Install the Termux:Boot companion APK if you want services to start automatically when Android boots. Without it, you must open Termux once after a reboot before services start.

## Enable and control `sshd`
1. Ensure your SSH public key lives in `$HOME/.ssh/authorized_keys` with `chmod 600` and `chmod 700` on the containing directory.
2. Start the daemon once to confirm everything works:
   ```bash
   sshd
   pkill sshd   # stop any manual instance before enabling runit
   ```
3. Kick off the runit supervisor and enable the bundled `sshd` service:
   ```bash
   service-daemon start
   sv-enable sshd
   sv up sshd
   ```
   This removes the `down` flag and asks runit to keep `sshd` alive under `/data/data/com.termux/files/usr/var/service/sshd`.
4. Verify status and test connectivity:
   ```bash
   sv status sshd
   tail -n 20 $PREFIX/var/log/sv/sshd/current
   ```
   From another machine: `ssh luis-tab-s8 hostname` (targets `[192.168.5.72]:8022`).

## PostgreSQL service (`postgresql`)
1. Initialize your PostgreSQL 14 data directory **inside the Ubuntu proot** (README §12 covers the steps). Example:
   ```bash
   proot-distro login ubuntu -- /usr/lib/postgresql/14/bin/initdb \
     -D /home/lyucra/postgres14-data \
     --auth-local=trust --auth-host=scram-sha-256
   ```
2. Start the supervisor **from the Termux host shell** (outside the Ubuntu proot), remove the `down` flag, and bring the service up. The run script defaults to major `14` in the `ubuntu` proot with the same bind mount as the `ubuntu` alias (`--bind /sdcard:/sdcard`); override `POSTGRES_MAJOR`, `POSTGRES_DATA_DIR`, `POSTGRES_PROOT_DISTRO`, `POSTGRES_PROOT_USER`, or `POSTGRES_PROOT_BIND` as needed:
   ```bash
   service-daemon start
   sv-enable postgresql
   sv up postgresql
   ```
   The run script execs `proot-distro login ubuntu -- /usr/lib/postgresql/14/bin/postgres -D /home/lyucra/postgres14-data` with `logging_collector=off` so logs flow into runit.
3. Check health and review logs (ensure no leftover host-level PostgreSQL processes are running, e.g. `pgrep postgres` should return nothing before enabling 14):
   ```bash
   sv status postgresql
   tail -n 40 $PREFIX/var/log/sv/postgresql/current
   ```
   If you see `data directory ... not found`, revisit step 1 or adjust the path in `var/service/postgresql/run`.

## Redis service (`redis`)
1. Install the Termux build of Redis so the binary matches the Android libc:
   ```bash
   pkg install redis
   ```
2. (Optional) Drop a custom config at `~/redis.conf`; otherwise the service starts Redis with sensible defaults (`--daemonize no --protected-mode yes --bind 127.0.0.1 --port 6379`).
3. Enable and start:
   ```bash
   service-daemon start
   sv-enable redis
   sv up redis
   ```
4. Validate status/logs and ping the instance:
   ```bash
   sv status redis
   tail -n 40 $PREFIX/var/log/sv/redis/current
   redis-cli ping
   ```

## Service management cheatsheet
- `service-daemon start|stop|status` — control the background `runsvdir` instance.
- `sv status <name>` — view status; `sv up|down|restart <name>` to control individual services.
- `sv-enable <name>` / `sv-disable <name>` — toggle the `down` flag (controls autostart).
- Logs live under `$PREFIX/var/log/sv/<service>/current` when the service ships with logging.

## Custom services
1. Create a directory under `$PREFIX/var/service/<name>/` containing an executable `run` script.
2. The `run` script should `exec` the long-running command (example below). Avoid daemonizing within the script; runit keeps it alive.
   ```bash
   #!/data/data/com.termux/files/usr/bin/sh
   exec /data/data/com.termux/files/usr/bin/python3 /data/data/com.termux/files/home/bin/my-daemon.py
   ```
3. Make the script executable: `chmod +x run`.
4. Termux automatically discovers the service; manage it with `sv` and the helpers above.

## Boot persistence
- With Termux:Boot installed and granted the “run scripts on boot” permission, create `~/.termux/boot/01-start-services.sh` containing `service-daemon start && sv up sshd`.
- Mark it executable (`chmod +x ~/.termux/boot/01-start-services.sh`).
- On the next Android boot, Termux:Boot triggers the script, bringing `sshd` (and any other enabled services) up automatically.

_Last updated: 2025-10-08._
