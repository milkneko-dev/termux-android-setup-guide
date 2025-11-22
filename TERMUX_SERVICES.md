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

### Preferred: native Termux Postgres (no proot)
1. Install Termux packages and init the cluster inside Termux:
   ```bash
   pkg install postgresql
   initdb -D ~/postgres-data --auth=scram-sha-256
   ```
2. Enable and start the bundled runit service (it uses the Termux `postgres` binary):
   ```bash
   service-daemon start
   sv-enable postgresql
   sv up postgresql
   ```
3. Validate:
   ```bash
   sv status postgresql
   tail -n 40 $PREFIX/var/log/sv/postgresql/current
   pg_isready -h 127.0.0.1 -p 5432
   ```
4. If you previously had a proot-backed Postgres on 5432, stop it first to avoid port conflicts: `sv down postgresql && pkill -f '/usr/lib/postgresql'` inside proot, then use the native service above.

### Legacy: proot-backed Postgres
Only keep this if you need glibc extensions. The run script execs `proot-distro login ubuntu -- /usr/lib/postgresql/14/bin/postgres ...` and logs to runit. Disable it if you move to the native Termux instance.

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
