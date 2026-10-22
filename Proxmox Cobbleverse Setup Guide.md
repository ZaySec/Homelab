>[!danger] AI formatted this guide

>[!hint] Redacted :)

> [!tldr] Definitive walkthrough for self-hosting the **COBBLEVERSE** (Cobblemon) server in a Proxmox LXC on the **Gnome** lab. Battle-tested — includes every fix from the original build.
> 
> - Host: Container LXC (Cobbleverse-Debian)
> - Pack: COBBLEVERSE — Fabric, MC `1.21.1`, no official server pack → built from the Modrinth `.mrpack`

---

# Golden rules (these prevent ~90% of the problems)

> [!warning] Read before doing anything
> 
> 1. **Run the server as `mc`, never as `root`.** The systemd service already runs as `mc`. If you ever launch `./start.sh` manually as root (e.g. for Chunky or to use the console), you MUST run `chown -R mc:mc /opt/cobbleverse` afterward, or the service later fails with `AccessDeniedException` on `level.dat`.
> 2. **Pick ONE launch method at a time — systemd OR manual tmux, never both.** Two instances = `session.lock: already locked`. For normal play, let systemd run it.
> 3. **JVM heap must be ≥2 GB below the container's RAM.** 8 GB heap in a 10 GB container OOM-kills the process (`[server exited]`). Container is now 16 GB, so 6–8 GB heap is safe.
> 4. **`server.properties` and `world/` only exist after the first launch.** Don't look for them before Step 4.
> 5. **Keep `online-mode=true`** while the server is reachable via playit (public tunnel).

---

# Sizing (current)

|Setting|Value|
|---|---|
|Template|`debian-13-standard` · **unprivileged**|
|Cores|`4`|
|RAM|`16384` MB|
|Swap|`2048` MB|
|Disk|`~63` GB|
|JVM heap|`-Xms6G -Xmx6G`|
|Network|`192.168.1.102/24` static|
|Start at boot|yes|

---

# 1. Base setup (inside the LXC, as root)

```shell
apt update && apt -y upgrade
apt -y install openjdk-21-jre-headless wget tmux
java -version                      # CHECKPOINT: must report 21.x

adduser --system --group --home /opt/cobbleverse mc
id mc                              # CHECKPOINT: must print uid=...(mc)

mkdir -p /opt/cobbleverse/data
ls -d /opt/cobbleverse/data        # CHECKPOINT: prints the path, no error
```

Don't continue until all three CHECKPOINTs pass.

---

# 2. Deploy the server with `mrpack-install`

```shell
cd /opt/cobbleverse/data
pwd                                # CHECKPOINT: must be /opt/cobbleverse/data (installing in the wrong dir was a real failure)

wget https://github.com/nothub/mrpack-install/releases/latest/download/mrpack-install-linux -O mrpack-install
chmod +x mrpack-install
./mrpack-install cobbleverse --server-dir . --server-file server.jar

ls *.jar                           # CHECKPOINT: server.jar present
```

> [!info] `libraries/` is missing right after this — that's normal `server.jar` is a tiny (~180 KB) Fabric launcher stub. The real Minecraft server + `libraries/`, `versions/`, `.fabric/` download on the **first launch** (Step 4). Don't panic.

> [!info]- If the Modrinth API route is blocked Download the `.mrpack` from `modrinth.com/modpack/cobbleverse` → Files, then `./mrpack-install ./COBBLEVERSE-<ver>.mrpack --server-dir . --server-file server.jar`.

---

# 3. EULA + start script

```shell
cd /opt/cobbleverse/data
echo "eula=true" > eula.txt

cat > start.sh <<'EOF'
#!/bin/bash
cd /opt/cobbleverse/data
java -Xms6G -Xmx6G \
  -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 \
  -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch \
  -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40 -XX:G1HeapRegionSize=8M \
  -XX:G1ReservePercent=20 -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 \
  -XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 \
  -XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 \
  -XX:+PerfDisableSharedMem -XX:MaxTenuringThreshold=1 \
  -Dusing.aikars.flags=https://mcflags.emc.gs -Daikars.new.flags=true \
  -jar server.jar nogui
EOF
chmod +x start.sh
```

---

# 4. First run — generates `server.properties` + world

```shell
cd /opt/cobbleverse/data
./start.sh                         # downloads server + libraries, generates the world
```

Wait for `Done (x.xxxs)! For help, type "help"`, then stop cleanly — type into that console:

```
stop
```

---

# 5. Configure `server.properties`

```shell
nano /opt/cobbleverse/data/server.properties
```

```ini
motd=Gnome COBBLEVERSE
max-players=8
online-mode=true            # keep true while exposed via playit
view-distance=8             # biggest TPS lever on the i7 — lower if it lags
simulation-distance=6
```

---

# 6. Hand ownership to `mc` (CRITICAL)

> [!warning] Do this every time after running as root The first run was as root, so the world files are root-owned. The service runs as `mc` and must own them or it dies with `AccessDeniedException: ./world/level.dat`.

```shell
chown -R mc:mc /opt/cobbleverse
ls -l /opt/cobbleverse/data/world/level.dat*   # CHECKPOINT: owner should be mc
```

---

# 7. Run it under systemd (survives reboots)

```shell
cat > /etc/systemd/system/cobbleverse.service <<'EOF'
[Unit]
Description=COBBLEVERSE Minecraft Server
After=network-online.target
Wants=network-online.target

[Service]
User=mc
Group=mc
WorkingDirectory=/opt/cobbleverse/data
ExecStart=/opt/cobbleverse/data/start.sh
Restart=on-failure
SuccessExitStatus=0 143
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now cobbleverse          # starts now AND on every boot
journalctl -u cobbleverse -f                 # watch for "Done"
```

`enable` is what makes it auto-recover after a reboot — the missing piece that caused the "playit up, server down" outage.

---

# 8. Expose to friends with playit

> [!info] What playit does Creates an **outbound** tunnel so friends connect via a public address — no port forwarding, and your home IP stays hidden. Free tier covers 3 tunnels. Dashboard: `playit.gg/account/agents`.

**Install** (Debian apt — adds a systemd unit on v0.15.0+):

```shell
# On playit.gg → Download, copy the "Debian Install Script (apt-get)" command and paste it.
# It adds the playit repo + key, then:
apt -y install playit
```

**Claim the agent to your account:**

```shell
playit
```

It prints a claim URL (`playit.gg/claim/...`). Open it in a browser, log in, approve and name the agent (e.g. `gnome-cobbleverse`).

**Create the tunnel** (dashboard → your agent → Add Tunnel):

- Type: **Minecraft Java**
- Region: Global Anycast (free)
- Local Address: `192.168.1.102` · Local Port: `25565` _(the LXC's own LAN IP, since playit runs on the same box)_

playit assigns a public address like `TEST.gl.joinmc.link` → share that with friends (no port needed; playit maps it).

**Keep it running 24/7** — same lesson as the server: use the service so it auto-starts on boot.

```shell
systemctl enable --now playit
```

_(Or run it in a tmux window: `tmux` → `playit` → detach with `Ctrl-b` then `d`. Note: tmux does NOT survive a reboot; the service does.)_

Connecting: **friends** use the public playit address; **you on the LAN** connect straight to `192.168.1.102` (skips the tunnel entirely).

---

# 9. Day-to-day operation

- **Start / stop / restart / status:** `systemctl start|stop|restart|status cobbleverse`
- **Live log:** `journalctl -u cobbleverse -f`
- **Console (to type server commands like `op`, `chunky`, `whitelist`):** the systemd service has no interactive console, so stop it and run in tmux:
    
    ```shell
    systemctl stop cobbleversetmux                                   # (or Ctrl-b c for a new window if already in tmux)cd /opt/cobbleverse/data && ./start.sh # this window is now the live console
    ```
    
    When finished: type `stop`, then **`chown -R mc:mc /opt/cobbleverse`** (you ran as root!), then `systemctl start cobbleverse`.
- **Never** run `./start.sh` while the service is active — that's the lock conflict.

> [!info]- tmux quick reference
> 
> - New window: `Ctrl-b` then `c` · Switch: `Ctrl-b` then `0`/`1`
> - Detach (leave running): `Ctrl-b` then `d` · Reattach: `tmux attach`
> - Run playit in one window, the server in another, when doing manual sessions.

---

# 10. First-time player setup (starters)

> [!info] Cobbleverse doesn't hand you a starter on spawn The starter + kit come through the **intro**, which is driven by Cobbleverse's bundled **resource pack**. If players spawn with nothing:
> 
> 1. Each client: **Options → Resource Packs** → ensure the **Cobbleverse** pack is active.
> 2. Known fix if the intro won't show: hover the pack, **hold T ~3 s**, click **Done**, rejoin.
> 3. Op fallback to just grant a starter: `/pokegive <player> bulbasaur` (tab-complete `/poke` for options). Bypasses the kit, so prefer the resource-pack fix.
> 4. Do this on a **calm server** (pregen finished/paused) — load can stop the intro firing.

---

# 11. Pre-generate the world (Chunky) — optional, reduces exploration lag

> [!info] Chunky is server-side only and removable — friends never need it
> 
> 1. Download **Chunky (Fabric, 1.21.1)** from `modrinth.com/mod/chunky` (right-click the Download button → copy link), then:
> 
> ```shell
> cd /opt/cobbleverse/data/mods
> wget -O Chunky.jar "<cdn.modrinth.com link>"
> chown mc:mc Chunky.jar     # ls -l: should be a few hundred KB, not a tiny error page
> ```
> 
> Fabric API is already in the pack — no need to add it. 2. Stop the service so it can't fight for the world, run interactively in tmux:
> 
> ```shell
> systemctl stop cobbleverse
> cd /opt/cobbleverse/data && ./start.sh
> ```
> 
> 2. After `Done`, in that console (set params, then start):
> 
> ```
> chunky world overworld
> chunky shape square
> chunky center 0 0
> chunky radius 5000
> chunky start
> ```
> 
> 3. Detach tmux (`Ctrl-b` then `d`). **Don't join while it runs** (max CPU = lag), keep an eye on `df -h`, don't reboot mid-task. Resume after interruption with `chunky continue`.
> 4. When done: `stop`, then `chown -R mc:mc /opt/cobbleverse` (you ran as root!), then `systemctl start cobbleverse`.

---

# 🔧 Troubleshooting reference (everything we hit)

|Symptom|Cause|Fix|
|---|---|---|
|`[server exited]` instantly|Heap too big for container RAM (OOM)|Heap ≤ RAM − 2 GB; lower `-Xmx` or raise container RAM|
|`status=217/USER`|`mc` user doesn't exist|Re-run `adduser ... mc` (Step 1)|
|`status=203/EXEC`|`start.sh` missing / not executable|Recreate it + `chmod +x` (Step 3)|
|`session.lock: already locked`|Two instances, or unclean reboot|Stop all: `systemctl stop cobbleverse; pkill -f server.jar`; if it lingers and **no** java runs: `rm world/session.lock`|
|`AccessDeniedException: level.dat`|Ran as root, files root-owned|`chown -R mc:mc /opt/cobbleverse`|
|playit: `ConnectionRefused (code 111)`|MC server not running|Start the service; confirm `Done` in the log|
|In-game: "Authentication servers are down"|No internet + `online-mode=true`|Temporarily set `online-mode=false` (⚠ different UUID = fresh char; re-enable after, never leave off on a public tunnel)|
|Spawn with no starter/kit|Intro/resource pack didn't fire|Enable Cobbleverse resource pack, T-trick, rejoin; or `/pokegive`|
|`Can't keep up! ... ticks behind`|One-time startup load (ignore) **or** TPS death|If recurring during play: lower `view`/`simulation-distance`, pregen with Chunky|
|`libraries/` missing after install|Downloads on first launch|Normal — run Step 4|
|Stuck falling / can't interact after join|Server under heavy load (pregen/lag)|Join only on a calm server; `/kill` if stuck mid-air|
|playit running but friends can't join|playit up, game server down|playit is its own service; check `systemctl status cobbleverse`|
