# Palworld Dedicated Server on Proxmox LXC

## Overview

This setup runs a **Palworld Dedicated Server directly inside an unprivileged Debian 12 LXC on Proxmox using SteamCMD**.

It does not require Docker or a full virtual machine.


> **Note:** Values such as `<BACKUP_SERVER_IP>`, `<BACKUP_HOSTNAME>`,
> `<WORLD_GUID>`, `YOUR_SERVER_PASSWORD`, and `YOUR_ADMIN_PASSWORD` are placeholders.
> Replace them with values from your own environment.


The LXC provides:

- Palworld Dedicated Server installation through SteamCMD
- automatic Palworld startup when the LXC boots
- automatic Steam update checks every 5 minutes
- automatic Palworld update installation when a new build is available
- a 5-minute in-game warning before update restarts
- a daily maintenance window at approximately 03:00 German local time
- a 5-minute in-game warning before daily maintenance
- an explicit world save before each controlled restart
- built-in Palworld rotating backups
- daily Debian security/package maintenance while Palworld is stopped
- health checks after maintenance and updates
- retry logic if Palworld does not start correctly
- timeout protection for APT, unattended-upgrades, SteamCMD, and systemd maintenance jobs
- disk-space checks before package maintenance and game updates
- a cooldown after failed Steam updates to prevent restart loops
- local Palworld REST API access for maintenance tasks
- automatic off-host backup mirroring every 30 minutes to a Raspberry Pi with a 4 TB SSD

The Palworld REST API is used only locally through `127.0.0.1` and should **not be exposed to the Internet**.

For normal player connections, the important public game port is:

```text
UDP 8211
```

---

# 1. Proxmox LXC configuration

Recommended configuration:

| Setting | Value |
|---|---|
| OS | Debian 12 amd64 |
| Container type | Unprivileged |
| Cores | 4 |
| RAM | 16384 MB |
| Swap | 4096 MB |
| Disk | 50 GB |
| Storage | SSD/NVMe recommended |
| Network bridge | `vmbr0` |
| IPv4 | DHCP reservation or static IP |
| Proxmox Firewall | enabled |
| Start at boot | enabled |
| Nesting | not required |

A static IP or DHCP reservation is recommended so the Palworld server always uses the same address.

---

# 2. Update Debian

Run as `root` inside the LXC:

```bash
apt update
apt full-upgrade -y
```

---

# 3. Enable Debian repositories required by SteamCMD

A typical Debian 12 Proxmox LXC initially contains:

```text
deb http://deb.debian.org/debian bookworm main contrib

deb http://deb.debian.org/debian bookworm-updates main contrib

deb http://security.debian.org bookworm-security main contrib
```

Back up the repository configuration:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

Enable `non-free` and `non-free-firmware`:

```bash
sed -i 's/ main contrib$/ main contrib non-free non-free-firmware/' /etc/apt/sources.list
```

Enable the i386 architecture required by SteamCMD:

```bash
dpkg --add-architecture i386
```

Refresh package lists:

```bash
apt update
```

---

# 4. Install required packages

```bash
apt install -y \
  steamcmd \
  curl \
  jq \
  ca-certificates \
  xdg-user-dirs \
  libcurl4 \
  locales \
  nano \
  unattended-upgrades \
  rsync \
  openssh-client
```

Test SteamCMD:

```bash
/usr/games/steamcmd +quit
```

---

# 5. Configure locale

Enable `en_US.UTF-8`:

```bash
sed -i '/^# *en_US.UTF-8 UTF-8/s/^# *//' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
```

Verify:

```bash
locale -a | grep -i en_US
locale
```

Expected output includes:

```text
en_US.utf8
LANG=en_US.UTF-8
```

---

# 6. Configure German local time

On an unprivileged LXC, `timedatectl set-timezone` may fail.

Set the timezone directly:

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
echo "Europe/Berlin" > /etc/timezone
```

Verify:

```bash
date
readlink -f /etc/localtime
```

Expected:

```text
/usr/share/zoneinfo/Europe/Berlin
```

CET/CEST changes are handled automatically.

---

# 7. Create the Palworld service account

Do not run the game server as `root`.

```bash
adduser --disabled-password --gecos "" palworld

mkdir -p /opt/palworld
chown palworld:palworld /opt/palworld
```

Verify:

```bash
id palworld
ls -ld /opt/palworld
```

---

# 8. Install the Palworld Dedicated Server

Palworld Dedicated Server Steam app ID:

```text
2394010
```

Install it as the `palworld` user:

```bash
runuser -u palworld -- env HOME=/home/palworld /usr/games/steamcmd \
  +force_install_dir /opt/palworld \
  +login anonymous \
  +app_update 2394010 validate \
  +quit
```

Verify:

```bash
ls -lah /opt/palworld
```

Important files/directories include:

```text
/opt/palworld/PalServer.sh
/opt/palworld/DefaultPalWorldSettings.ini
/opt/palworld/Pal/
/opt/palworld/steamapps/
```

---

# 9. Start Palworld manually once

```bash
runuser -u palworld -- env HOME=/home/palworld \
  bash -c 'cd /opt/palworld && ./PalServer.sh'
```

A successful startup should eventually show something similar to:

```text
Game version is ...
Running Palworld dedicated server on :8211
```

Stop it with:

```text
Ctrl+C
```

The first launch creates the required configuration structure.

---

# 10. Create the Palworld configuration

Copy the default configuration while the server is stopped:

```bash
cp /opt/palworld/DefaultPalWorldSettings.ini \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini

chown palworld:palworld \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Main configuration file:

```text
/opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Edit it:

```bash
nano /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Useful settings include:

```ini
ServerName="My Palworld Server"
ServerDescription="Welcome to our Palworld server!"
ServerPassword="YOUR_SERVER_PASSWORD"
AdminPassword="YOUR_ADMIN_PASSWORD"
ServerPlayerMaxNum=32
bIsUseBackupSaveData=True
RESTAPIEnabled=True
RESTAPIPort=8212
```

> Important: Edit `PalWorldSettings.ini` while Palworld is stopped. A running server may write in-memory values back to the file during shutdown.

---

# 11. Built-in Palworld backups

Ensure this option is enabled:

```ini
bIsUseBackupSaveData=True
```

Verify:

```bash
grep -o 'bIsUseBackupSaveData=[A-Za-z]*' \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Expected:

```text
bIsUseBackupSaveData=True
```

World and built-in backup data live under:

```text
/opt/palworld/Pal/Saved/
```

The active backup directory for the current world in this setup is:

```text
/opt/palworld/Pal/Saved/SaveGames/0/<WORLD_GUID>/backup
```

> The GUID directory belongs to the current Palworld world. If a completely new world is created, verify the path again before relying on automated remote backups.

Find the active backup directory with:

```bash
find /opt/palworld/Pal/Saved \
  -type d \
  \( -iname 'backup' -o -iname 'backups' \) \
  -print
```

---

# 12. Enable the Palworld REST API

The REST API is used by maintenance automation for:

- in-game announcements
- explicit world saves
- server information

Stop the server before changing the setting:

```bash
systemctl stop palworld.service 2>/dev/null || true
```

Enable the REST API:

```bash
sed -i 's/RESTAPIEnabled=False/RESTAPIEnabled=True/' \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Verify:

```bash
grep -o 'RESTAPIEnabled=[A-Za-z]*' \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini

grep -o 'RESTAPIPort=[0-9]*' \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini
```

Expected:

```text
RESTAPIEnabled=True
RESTAPIPort=8212
```

Do **not** forward TCP 8212 through the router.

The automation uses only:

```text
http://127.0.0.1:8212
```

---

# 13. Create the Palworld systemd service

Create:

```bash
nano /etc/systemd/system/palworld.service
```

Contents:

```ini
[Unit]
Description=Palworld Dedicated Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=palworld
Group=palworld
Environment=HOME=/home/palworld
WorkingDirectory=/opt/palworld
ExecStart=/opt/palworld/PalServer.sh
Restart=on-failure
RestartSec=10
KillSignal=SIGINT
TimeoutStopSec=120
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

Reload and enable:

```bash
systemctl daemon-reload
systemctl enable palworld.service
systemctl start palworld.service
```

Verify:

```bash
systemctl status palworld.service --no-pager
```

Check UDP 8211:

```bash
ss -lunp | grep 8211
```

---

# 14. Test the REST API

Read the admin password directly from the Palworld configuration:

```bash
ADMIN_PASSWORD=$(perl -ne 'if (/AdminPassword="([^"]*)"/) { print $1; exit }' \
  /opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini)
```

Server information:

```bash
curl -sS \
  -u "admin:${ADMIN_PASSWORD}" \
  -H "Accept: application/json" \
  http://127.0.0.1:8212/v1/api/info | jq
```

Test an in-game announcement:

```bash
curl -sS \
  -u "admin:${ADMIN_PASSWORD}" \
  -H "Content-Type: application/json" \
  -X POST \
  http://127.0.0.1:8212/v1/api/announce \
  --data '{"message":"Test: server automation is working."}'
```

Test an explicit world save:

```bash
curl -i -sS \
  -u "admin:${ADMIN_PASSWORD}" \
  --data '' \
  http://127.0.0.1:8212/v1/api/save
```

Expected:

```text
HTTP/1.1 200
```

---

# 15. Test Steam update detection

Read the installed Steam build ID:

```bash
BUILDID=$(awk -F'"' '/"buildid"/ {print $4; exit}' \
  /opt/palworld/steamapps/appmanifest_2394010.acf)

echo "Installed build ID: $BUILDID"
```

Check Steam:

```bash
curl -fsS \
  "https://api.steampowered.com/ISteamApps/UpToDateCheck/v1/?appid=2394010&version=${BUILDID}" \
  | jq
```

A current installation should return something similar to:

```json
{
  "response": {
    "success": true,
    "up_to_date": true,
    "version_is_listable": true
  }
}
```

---

# 16. Configure unattended Debian updates

Install if not already present:

```bash
apt install -y unattended-upgrades
```

Dry-run test:

```bash
unattended-upgrade --dry-run --debug
```

This setup deliberately does **not** run an unattended daily `apt full-upgrade`.

---

# 17. Disable Debian's default APT timers

```bash
systemctl disable --now apt-daily.timer
systemctl disable --now apt-daily-upgrade.timer
```

The maintenance script performs:

```text
apt-get update
unattended-upgrade
```

inside the controlled daily maintenance window.

---

# 18. Hardened Palworld maintenance script

Create:

```bash
nano /usr/local/sbin/palworld-maintenance
```

Contents:

```bash
#!/bin/bash

set -u

SERVICE="palworld.service"
INSTALL_DIR="/opt/palworld"
CONFIG="/opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini"
STEAMCMD="/usr/games/steamcmd"
APPID="2394010"
API="http://127.0.0.1:8212/v1/api"
GAME_PORT="8211"

MIN_STEAM_FREE_GB=15
MIN_APT_FREE_GB=2
START_TIMEOUT=180
UPDATE_FAILURE_COOLDOWN=1800

STATE_DIR="/var/lib/palworld-maintenance"
UPDATE_FAILURE_FILE="${STATE_DIR}/last-update-failure"

SERVER_STOPPED_BY_MAINTENANCE=0

mkdir -p "$STATE_DIR"
chmod 700 "$STATE_DIR"

exec 9>/run/lock/palworld-maintenance.lock

if ! flock -n 9; then
    logger -t palworld-maintenance \
      "Another maintenance task is already running - skipping."
    exit 0
fi

ADMIN_PASSWORD=$(perl -ne \
  'if (/AdminPassword="([^"]*)"/) { print $1; exit }' \
  "$CONFIG")

if [ -z "$ADMIN_PASSWORD" ]; then
    logger -t palworld-maintenance \
      "ERROR: Could not read AdminPassword."
    exit 1
fi

announce()
{
    local MESSAGE="$1"
    local JSON

    JSON=$(jq -nc --arg message "$MESSAGE" '{message:$message}')

    curl -fsS \
        --max-time 10 \
        -u "admin:${ADMIN_PASSWORD}" \
        -H "Content-Type: application/json" \
        -X POST \
        "${API}/announce" \
        --data "$JSON" \
        >/dev/null
}

save_world()
{
    curl -fsS \
        --max-time 30 \
        -u "admin:${ADMIN_PASSWORD}" \
        --data '' \
        "${API}/save" \
        >/dev/null
}

game_port_is_listening()
{
    ss -lunH | awk -v p=":${GAME_PORT}" \
      '$4 ~ (p "$") {found=1} END {exit(found ? 0 : 1)}'
}

wait_for_server()
{
    local ELAPSED=0

    while [ "$ELAPSED" -lt "$START_TIMEOUT" ]; do

        if systemctl is-active --quiet "$SERVICE" \
           && game_port_is_listening; then

            logger -t palworld-maintenance \
              "Palworld health check passed: service active and UDP ${GAME_PORT} is listening."

            return 0
        fi

        sleep 2
        ELAPSED=$((ELAPSED + 2))
    done

    logger -t palworld-maintenance \
      "ERROR: Palworld did not become healthy within ${START_TIMEOUT} seconds."

    return 1
}

start_server()
{
    logger -t palworld-maintenance \
      "Starting Palworld."

    systemctl reset-failed "$SERVICE" >/dev/null 2>&1 || true
    systemctl start "$SERVICE" || true

    if wait_for_server; then
        SERVER_STOPPED_BY_MAINTENANCE=0
        return 0
    fi

    logger -t palworld-maintenance \
      "WARNING: First Palworld start attempt failed. Retrying once."

    systemctl stop "$SERVICE" >/dev/null 2>&1 || true
    sleep 5

    systemctl reset-failed "$SERVICE" >/dev/null 2>&1 || true
    systemctl start "$SERVICE" || true

    if wait_for_server; then
        SERVER_STOPPED_BY_MAINTENANCE=0
        return 0
    fi

    logger -t palworld-maintenance \
      "ERROR: Palworld failed to become healthy after the retry."

    return 1
}

check_free_space()
{
    local PATH_TO_CHECK="$1"
    local MIN_GB="$2"
    local PURPOSE="$3"

    local FREE_KB
    local MIN_KB
    local FREE_GB

    FREE_KB=$(df -Pk "$PATH_TO_CHECK" 2>/dev/null \
      | awk 'NR==2 {print $4}')

    if ! [[ "$FREE_KB" =~ ^[0-9]+$ ]]; then
        logger -t palworld-maintenance \
          "ERROR: Could not determine free disk space for ${PATH_TO_CHECK}."
        return 1
    fi

    MIN_KB=$((MIN_GB * 1024 * 1024))

    FREE_GB=$(awk -v kb="$FREE_KB" \
      'BEGIN {printf "%.1f", kb / 1024 / 1024}')

    if [ "$FREE_KB" -lt "$MIN_KB" ]; then
        logger -t palworld-maintenance \
          "ERROR: Not enough free disk space for ${PURPOSE}. Free: ${FREE_GB} GiB, required: ${MIN_GB} GiB."
        return 1
    fi

    logger -t palworld-maintenance \
      "Disk space check for ${PURPOSE}: ${FREE_GB} GiB free."

    return 0
}

stop_server_with_warning()
{
    local MESSAGE="$1"

    if ! systemctl is-active --quiet "$SERVICE"; then
        logger -t palworld-maintenance \
          "Palworld is already stopped. No in-game warning can be sent."
        return 0
    fi

    logger -t palworld-maintenance \
      "Sending 5-minute in-game warning."

    if ! announce "$MESSAGE"; then
        logger -t palworld-maintenance \
          "ERROR: Could not send the in-game warning. Maintenance aborted."
        return 1
    fi

    sleep 295

    if systemctl is-active --quiet "$SERVICE"; then
        logger -t palworld-maintenance \
          "Saving world."

        if ! save_world; then
            logger -t palworld-maintenance \
              "WARNING: REST world save failed. Maintenance will continue."
        fi

        sleep 5
    fi

    logger -t palworld-maintenance \
      "Stopping Palworld."

    SERVER_STOPPED_BY_MAINTENANCE=1

    if ! systemctl stop "$SERVICE"; then
        logger -t palworld-maintenance \
          "ERROR: systemctl stop failed."

        if systemctl is-active --quiet "$SERVICE"; then
            SERVER_STOPPED_BY_MAINTENANCE=0
        fi

        return 1
    fi

    return 0
}

debian_updates()
{
    if ! check_free_space "/" "$MIN_APT_FREE_GB" "Debian package maintenance"; then
        logger -t palworld-maintenance \
          "WARNING: Debian updates skipped because of insufficient disk space."
        return 1
    fi

    logger -t palworld-maintenance \
      "Updating Debian package lists."

    if ! timeout \
        --signal=TERM \
        --kill-after=30s \
        10m \
        apt-get \
          -o Acquire::Retries=3 \
          -o Acquire::http::Timeout=30 \
          -o Acquire::https::Timeout=30 \
          update; then

        logger -t palworld-maintenance \
          "WARNING: apt-get update failed or timed out. Palworld will still be restarted."
        return 1
    fi

    logger -t palworld-maintenance \
      "Installing unattended Debian updates."

    if ! timeout \
        --signal=INT \
        --kill-after=5m \
        45m \
        env DEBIAN_FRONTEND=noninteractive \
        unattended-upgrade \
        --minimal-upgrade-steps; then

        logger -t palworld-maintenance \
          "WARNING: unattended-upgrade failed or exceeded its time limit. Palworld will still be restarted."
        return 1
    fi

    logger -t palworld-maintenance \
      "Debian package maintenance completed."

    return 0
}

update_failure_in_cooldown()
{
    local NOW
    local LAST
    local AGE
    local REMAINING

    [ -r "$UPDATE_FAILURE_FILE" ] || return 1

    LAST=$(cat "$UPDATE_FAILURE_FILE" 2>/dev/null || true)

    if ! [[ "$LAST" =~ ^[0-9]+$ ]]; then
        rm -f "$UPDATE_FAILURE_FILE"
        return 1
    fi

    NOW=$(date +%s)
    AGE=$((NOW - LAST))

    if [ "$AGE" -lt "$UPDATE_FAILURE_COOLDOWN" ]; then
        REMAINING=$((UPDATE_FAILURE_COOLDOWN - AGE))

        logger -t palworld-maintenance \
          "Previous Palworld update attempt failed. Retry cooldown active for another ${REMAINING} seconds."

        return 0
    fi

    rm -f "$UPDATE_FAILURE_FILE"
    return 1
}

mark_update_failure()
{
    date +%s > "$UPDATE_FAILURE_FILE"
}

clear_update_failure()
{
    rm -f "$UPDATE_FAILURE_FILE"
}

cleanup()
{
    local RC=$?

    trap - EXIT

    if [ "$SERVER_STOPPED_BY_MAINTENANCE" -eq 1 ]; then
        logger -t palworld-maintenance \
          "Maintenance exited while Palworld was stopped. Attempting emergency server start."

        start_server || true
    fi

    exit "$RC"
}

trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
trap 'exit 129' HUP

case "${1:-}" in

    daily)

        logger -t palworld-maintenance \
          "Daily maintenance started."

        if ! stop_server_with_warning \
          "Daily maintenance starts in 5 minutes. The server may be offline for several minutes."; then
            exit 1
        fi

        logger -t palworld-maintenance \
          "Running Debian package maintenance."

        debian_updates || true

        if ! start_server; then
            logger -t palworld-maintenance \
              "ERROR: Daily maintenance completed, but Palworld could not be started successfully."
            exit 1
        fi

        logger -t palworld-maintenance \
          "Daily maintenance completed successfully."

        ;;

    update)

        MANIFEST="${INSTALL_DIR}/steamapps/appmanifest_${APPID}.acf"

        BUILDID=$(awk -F'"' \
          '/"buildid"/ {print $4; exit}' \
          "$MANIFEST")

        if [ -z "$BUILDID" ]; then
            logger -t palworld-maintenance \
              "ERROR: Could not read local Steam build ID."
            exit 1
        fi

        RESPONSE=$(curl -fsS \
            --max-time 15 \
            "https://api.steampowered.com/ISteamApps/UpToDateCheck/v1/?appid=${APPID}&version=${BUILDID}") || {
                logger -t palworld-maintenance \
                  "Steam update check failed."
                exit 1
            }

        SUCCESS=$(echo "$RESPONSE" \
          | jq -r '.response.success // false')

        UP_TO_DATE=$(echo "$RESPONSE" \
          | jq -r '.response.up_to_date // empty')

        if [ "$SUCCESS" != "true" ]; then
            logger -t palworld-maintenance \
              "Steam update API did not report a successful check."
            exit 1
        fi

        if [ "$UP_TO_DATE" = "true" ]; then
            clear_update_failure
            exit 0
        fi

        if [ "$UP_TO_DATE" != "false" ]; then
            logger -t palworld-maintenance \
              "Invalid response from Steam."
            exit 1
        fi

        if update_failure_in_cooldown; then
            exit 0
        fi

        if ! check_free_space "$INSTALL_DIR" "$MIN_STEAM_FREE_GB" "Palworld update"; then
            logger -t palworld-maintenance \
              "Palworld update postponed. The running server will not be stopped."
            exit 1
        fi

        logger -t palworld-maintenance \
          "Palworld update found. Installed build ID: ${BUILDID}"

        if ! stop_server_with_warning \
          "Palworld update found. Server restart in 5 minutes."; then
            exit 1
        fi

        logger -t palworld-maintenance \
          "Installing Palworld update."

        timeout \
          --signal=TERM \
          --kill-after=5m \
          90m \
          runuser -u palworld -- env HOME=/home/palworld \
            "$STEAMCMD" \
            +force_install_dir "$INSTALL_DIR" \
            +login anonymous \
            +app_update "$APPID" validate \
            +quit

        UPDATE_RESULT=$?

        if ! start_server; then
            mark_update_failure

            logger -t palworld-maintenance \
              "ERROR: Palworld update process finished, but the server could not be started successfully."
            exit 1
        fi

        if [ "$UPDATE_RESULT" -ne 0 ]; then
            mark_update_failure

            logger -t palworld-maintenance \
              "ERROR: SteamCMD update failed or timed out with exit code ${UPDATE_RESULT}. Palworld was started again. Another disruptive update attempt will be delayed for 30 minutes."
            exit 1
        fi

        NEW_BUILDID=$(awk -F'"' \
          '/"buildid"/ {print $4; exit}' \
          "$MANIFEST")

        if [ -z "$NEW_BUILDID" ]; then
            mark_update_failure
            logger -t palworld-maintenance \
              "ERROR: Update finished but the new local Steam build ID could not be read."
            exit 1
        fi

        VERIFY_RESPONSE=$(curl -fsS \
            --max-time 15 \
            "https://api.steampowered.com/ISteamApps/UpToDateCheck/v1/?appid=${APPID}&version=${NEW_BUILDID}") || {
                mark_update_failure
                logger -t palworld-maintenance \
                  "WARNING: Update installed, but post-update Steam verification failed. Retry cooldown enabled."
                exit 1
            }

        VERIFY_SUCCESS=$(echo "$VERIFY_RESPONSE" \
          | jq -r '.response.success // false')

        VERIFY_UP_TO_DATE=$(echo "$VERIFY_RESPONSE" \
          | jq -r '.response.up_to_date // empty')

        if [ "$VERIFY_SUCCESS" != "true" ] \
           || [ "$VERIFY_UP_TO_DATE" != "true" ]; then

            mark_update_failure

            logger -t palworld-maintenance \
              "WARNING: SteamCMD completed, but Steam still reports build ${NEW_BUILDID} as not current. Another disruptive update attempt will be delayed for 30 minutes."

            exit 1
        fi

        clear_update_failure

        logger -t palworld-maintenance \
          "Palworld update completed successfully. New build ID: ${NEW_BUILDID}"

        ;;

    *)

        echo "Usage: $0 {daily|update}"
        exit 2

        ;;
esac
```

Set permissions:

```bash
chmod 750 /usr/local/sbin/palworld-maintenance
```

Validate:

```bash
bash -n /usr/local/sbin/palworld-maintenance
```

---

# 19. Maintenance systemd service

```bash
cat > /etc/systemd/system/palworld-maintenance@.service <<'EOF'
[Unit]
Description=Palworld Maintenance (%i)
After=network-online.target palworld.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/palworld-maintenance %i
TimeoutStartSec=2h
TimeoutStopSec=5min
EOF
```

Reload:

```bash
systemctl daemon-reload
```

---

# 20. Daily maintenance timer

```bash
cat > /etc/systemd/system/palworld-daily.timer <<'EOF'
[Unit]
Description=Daily Palworld Maintenance

[Timer]
OnCalendar=*-*-* 02:55:00 Europe/Berlin
AccuracySec=1s
Unit=palworld-maintenance@daily.service

[Install]
WantedBy=timers.target
EOF
```

Sequence:

```text
02:55:00  In-game warning
02:59:55  Explicit world save
03:00:00  Palworld stops
03:00:xx  apt-get update
           unattended-upgrade
           Palworld starts
           health check verifies service + UDP 8211
```

---

# 21. Steam update timer

```bash
cat > /etc/systemd/system/palworld-update.timer <<'EOF'
[Unit]
Description=Palworld Update Check

[Timer]
OnBootSec=1min
OnCalendar=*-*-* *:0/5:00
AccuracySec=1s
Unit=palworld-maintenance@update.service

[Install]
WantedBy=timers.target
EOF
```

Enable:

```bash
systemctl daemon-reload
systemctl enable --now palworld-daily.timer
systemctl enable --now palworld-update.timer
```

Verify:

```bash
systemctl list-timers --all | grep -E 'palworld-(daily|update)'
```

---

# 22. Test maintenance

Update check:

```bash
systemctl start palworld-maintenance@update.service
systemctl status palworld-maintenance@update.service --no-pager
systemctl is-active palworld.service
```

Full daily maintenance test:

```bash
systemctl start palworld-maintenance@daily.service
```

After completion:

```bash
systemctl status palworld-maintenance@daily.service --no-pager
systemctl is-active palworld.service
ss -lunp | grep 8211
```

Logs:

```bash
journalctl -t palworld-maintenance -n 100 --no-pager
```

---

# 23. Raspberry Pi off-host backup server

Remote target used in this setup:

```text
Hostname: <BACKUP_HOSTNAME>
IP:       <BACKUP_SERVER_IP>
Storage:  4 TB NVMe SSD
Mount:    /srv/backups
Target:   /srv/backups/palworld
```

Verify storage on the Raspberry Pi:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
df -hT /srv/backups
findmnt /srv/backups
```

The SSD should be persistently mounted via `/etc/fstab`.

---

# 24. Dedicated backup user on the Raspberry Pi

On `<BACKUP_HOSTNAME>`:

```bash
sudo adduser --disabled-password --gecos "" palworldbackup
sudo usermod -aG backupgrp palworldbackup
```

Prepare target directory:

```bash
sudo mkdir -p /srv/backups/palworld
sudo chown palworldbackup:backupgrp /srv/backups/palworld
sudo chmod 2770 /srv/backups/palworld
```

Verify:

```bash
id palworldbackup
ls -ld /srv/backups/palworld
```

Write test:

```bash
sudo -u palworldbackup touch /srv/backups/palworld/.write-test
sudo -u palworldbackup rm /srv/backups/palworld/.write-test
```

---

# 25. Dedicated SSH key for backups

On the Palworld LXC as `root`:

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh

ssh-keygen \
  -t ed25519 \
  -f /root/.ssh/palworld_backup_ed25519 \
  -N '' \
  -C 'palworld-backup@palworld'
```

Show public key:

```bash
cat /root/.ssh/palworld_backup_ed25519.pub
```

On `<BACKUP_HOSTNAME>`:

```bash
sudo install -d \
  -m 700 \
  -o palworldbackup \
  -g palworldbackup \
  /home/palworldbackup/.ssh
```

Edit:

```bash
sudo nano /home/palworldbackup/.ssh/authorized_keys
```

Paste the public key, then:

```bash
sudo chown palworldbackup:palworldbackup \
  /home/palworldbackup/.ssh/authorized_keys

sudo chmod 600 \
  /home/palworldbackup/.ssh/authorized_keys
```

Test from the Palworld LXC:

```bash
ssh \
  -i /root/.ssh/palworld_backup_ed25519 \
  -o BatchMode=yes \
  palworldbackup@<BACKUP_SERVER_IP> \
  'hostname && test -w /srv/backups/palworld && echo BACKUP_TARGET_OK'
```

Expected:

```text
<BACKUP_HOSTNAME>
BACKUP_TARGET_OK
```

---

# 26. rsync dry run

```bash
BACKUP_SOURCE="/opt/palworld/Pal/Saved/SaveGames/0/<WORLD_GUID>/backup"

rsync \
  -a \
  --delete \
  --dry-run \
  --itemize-changes \
  --stats \
  -e "ssh -i /root/.ssh/palworld_backup_ed25519 -o BatchMode=yes" \
  "${BACKUP_SOURCE}/" \
  "palworldbackup@<BACKUP_SERVER_IP>:/srv/backups/palworld/"
```

The trailing slash in `${BACKUP_SOURCE}/` is intentional.

---

# 27. First real rsync

```bash
BACKUP_SOURCE="/opt/palworld/Pal/Saved/SaveGames/0/<WORLD_GUID>/backup"

rsync \
  -a \
  --no-owner \
  --no-group \
  --delete-delay \
  --delay-updates \
  --stats \
  --timeout=60 \
  -e "ssh -i /root/.ssh/palworld_backup_ed25519 -o BatchMode=yes -o ConnectTimeout=10 -o ServerAliveInterval=15 -o ServerAliveCountMax=3" \
  "${BACKUP_SOURCE}/" \
  "palworldbackup@<BACKUP_SERVER_IP>:/srv/backups/palworld/"
```

Verify remotely:

```bash
ssh \
  -i /root/.ssh/palworld_backup_ed25519 \
  -o BatchMode=yes \
  palworldbackup@<BACKUP_SERVER_IP> \
  'du -sh /srv/backups/palworld && find /srv/backups/palworld -type f | wc -l'
```

The remote side intentionally mirrors Palworld's own backup rotation.

---

# 28. Automatic backup sync script

Create:

```bash
nano /usr/local/sbin/palworld-backup-sync
```

Contents:

```bash
#!/bin/bash

set -u

SOURCE="/opt/palworld/Pal/Saved/SaveGames/0/<WORLD_GUID>/backup"
REMOTE="palworldbackup@<BACKUP_SERVER_IP>"
DEST="/srv/backups/palworld/"
SSH_KEY="/root/.ssh/palworld_backup_ed25519"

exec 9>/run/lock/palworld-backup-sync.lock

if ! flock -n 9; then
    logger -t palworld-backup \
      "Another backup sync is already running - skipping."
    exit 0
fi

if [ ! -d "$SOURCE" ]; then
    logger -t palworld-backup \
      "ERROR: Palworld backup source does not exist: $SOURCE"
    exit 1
fi

if ! ssh \
    -i "$SSH_KEY" \
    -o BatchMode=yes \
    -o ConnectTimeout=10 \
    -o ServerAliveInterval=15 \
    -o ServerAliveCountMax=3 \
    "$REMOTE" \
    'mountpoint -q /srv/backups && test -w /srv/backups/palworld'; then

    logger -t palworld-backup \
      "ERROR: Backup server unavailable, backup filesystem not mounted, or destination not writable."

    exit 1
fi

logger -t palworld-backup \
  "Starting Palworld backup synchronization."

if rsync \
    -a \
    --no-owner \
    --no-group \
    --delete-delay \
    --delay-updates \
    --timeout=60 \
    -e "ssh -i ${SSH_KEY} -o BatchMode=yes -o ConnectTimeout=10 -o ServerAliveInterval=15 -o ServerAliveCountMax=3" \
    "${SOURCE}/" \
    "${REMOTE}:${DEST}"; then

    logger -t palworld-backup \
      "Palworld backup synchronization completed successfully."

    exit 0
else
    RESULT=$?

    logger -t palworld-backup \
      "ERROR: Palworld backup synchronization failed with exit code ${RESULT}."

    exit "$RESULT"
fi
```

Set permissions:

```bash
chmod 750 /usr/local/sbin/palworld-backup-sync
```

Validate:

```bash
bash -n /usr/local/sbin/palworld-backup-sync
```

---

# 29. Backup sync systemd service

```bash
cat > /etc/systemd/system/palworld-backup-sync.service <<'EOF'
[Unit]
Description=Palworld Backup Sync to <BACKUP_HOSTNAME>
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/palworld-backup-sync
TimeoutStartSec=10min
EOF
```

---

# 30. 30-minute backup timer

```bash
cat > /etc/systemd/system/palworld-backup-sync.timer <<'EOF'
[Unit]
Description=Palworld Backup Sync every 30 minutes

[Timer]
OnBootSec=5min
OnCalendar=*-*-* *:00,30:00
AccuracySec=30s
Persistent=true
Unit=palworld-backup-sync.service

[Install]
WantedBy=timers.target
EOF
```

Enable:

```bash
systemctl daemon-reload
systemctl enable --now palworld-backup-sync.timer
```

Verify:

```bash
systemctl list-timers --all | grep palworld-backup
```

---

# 31. Test backup synchronization

```bash
systemctl start palworld-backup-sync.service
```

Check:

```bash
systemctl status palworld-backup-sync.service --no-pager
```

Expected:

```text
status=0/SUCCESS
Active: inactive (dead)
```

This is normal for a `Type=oneshot` service.

Logs:

```bash
journalctl -t palworld-backup -n 50 --no-pager
```

Expected:

```text
Starting Palworld backup synchronization.
Palworld backup synchronization completed successfully.
```

---

# 32. Backup safety behavior

Before every sync, the script verifies:

```text
Raspberry Pi reachable
        |
/srv/backups actually mounted
        |
/srv/backups/palworld writable
        |
rsync starts
```

This prevents a missing 4 TB SSD mount from causing backup files to be written to the Raspberry Pi's root filesystem.

The sync uses:

```text
--delete-delay
```

so files removed by Palworld's own rotation are also removed from the remote mirror.

It also uses:

```text
--delay-updates
```

to delay final placement of updated files until the transfer is near completion.

---

# 33. Useful administration commands

## Palworld status

```bash
systemctl status palworld.service --no-pager
```

## Start

```bash
systemctl start palworld.service
```

## Stop

```bash
systemctl stop palworld.service
```

## Immediate restart

```bash
systemctl restart palworld.service
```

## Controlled 5-minute maintenance restart

```bash
systemctl start palworld-maintenance@daily.service
```

## Palworld live logs

```bash
journalctl -u palworld.service -f
```

## Maintenance logs

```bash
journalctl -t palworld-maintenance -n 100 --no-pager
```

## Backup logs

```bash
journalctl -t palworld-backup -n 100 --no-pager
```

## Live backup logs

```bash
journalctl -t palworld-backup -f
```

## Show all Palworld timers

```bash
systemctl list-timers --all | grep palworld
```

## Check game port

```bash
ss -lunp | grep 8211
```

## Check REST API

```bash
ss -ltnp | grep 8212
```

## Installed Steam build ID

```bash
awk -F'"' '/"buildid"/ {print $4; exit}' \
  /opt/palworld/steamapps/appmanifest_2394010.acf
```

## Manually trigger Steam update check

```bash
systemctl start palworld-maintenance@update.service
```

## Manually trigger backup sync

```bash
systemctl start palworld-backup-sync.service
```

## Check backup service

```bash
systemctl status palworld-backup-sync.service --no-pager
```

## Remote backup size and file count

```bash
ssh \
  -i /root/.ssh/palworld_backup_ed25519 \
  -o BatchMode=yes \
  palworldbackup@<BACKUP_SERVER_IP> \
  'du -sh /srv/backups/palworld && find /srv/backups/palworld -type f | wc -l'
```

## Check remote SSD mount

```bash
ssh \
  -i /root/.ssh/palworld_backup_ed25519 \
  -o BatchMode=yes \
  palworldbackup@<BACKUP_SERVER_IP> \
  'mountpoint /srv/backups'
```

## Disable backup timer

```bash
systemctl disable --now palworld-backup-sync.timer
```

## Enable backup timer

```bash
systemctl enable --now palworld-backup-sync.timer
```

## Disable update timer

```bash
systemctl disable --now palworld-update.timer
```

## Enable update timer

```bash
systemctl enable --now palworld-update.timer
```

## Disable daily maintenance

```bash
systemctl disable --now palworld-daily.timer
```

## Enable daily maintenance

```bash
systemctl enable --now palworld-daily.timer
```

## Validate maintenance script

```bash
bash -n /usr/local/sbin/palworld-maintenance
```

## Validate backup script

```bash
bash -n /usr/local/sbin/palworld-backup-sync
```

---

# 34. Important paths

| Purpose | Path |
|---|---|
| Palworld installation | `/opt/palworld` |
| Startup script | `/opt/palworld/PalServer.sh` |
| Main configuration | `/opt/palworld/Pal/Saved/Config/LinuxServer/PalWorldSettings.ini` |
| Save data | `/opt/palworld/Pal/Saved` |
| Current Palworld backup directory | `/opt/palworld/Pal/Saved/SaveGames/0/<WORLD_GUID>/backup` |
| Steam manifest | `/opt/palworld/steamapps/appmanifest_2394010.acf` |
| Palworld service | `/etc/systemd/system/palworld.service` |
| Maintenance script | `/usr/local/sbin/palworld-maintenance` |
| Maintenance service | `/etc/systemd/system/palworld-maintenance@.service` |
| Update timer | `/etc/systemd/system/palworld-update.timer` |
| Daily timer | `/etc/systemd/system/palworld-daily.timer` |
| Backup sync script | `/usr/local/sbin/palworld-backup-sync` |
| Backup sync service | `/etc/systemd/system/palworld-backup-sync.service` |
| Backup sync timer | `/etc/systemd/system/palworld-backup-sync.timer` |
| Backup SSH key | `/root/.ssh/palworld_backup_ed25519` |
| Remote backup target | `/srv/backups/palworld` |

---

# 35. Final architecture

```text
Proxmox
└── Debian 12 unprivileged LXC
    │
    ├── SteamCMD
    │
    ├── Palworld Dedicated Server
    │   └── UDP 8211
    │
    ├── Local REST API
    │   └── TCP 8212
    │
    ├── palworld.service
    │
    ├── palworld-update.timer
    │   └── every 5 minutes
    │       ├── check Steam build
    │       ├── if current: exit
    │       └── if update:
    │           ├── disk-space check
    │           ├── 5-minute warning
    │           ├── save world
    │           ├── stop server
    │           ├── SteamCMD update with timeout
    │           ├── restart
    │           ├── health check
    │           ├── retry once
    │           └── 30-minute cooldown after failure
    │
    ├── palworld-daily.timer
    │   └── 02:55 Europe/Berlin
    │       ├── 5-minute warning
    │       ├── save world
    │       ├── stop at ~03:00
    │       ├── apt-get update
    │       ├── unattended-upgrade
    │       ├── restart
    │       └── health check + retry
    │
    ├── Palworld built-in rotating backups
    │
    └── palworld-backup-sync.timer
        └── every 30 minutes
            ├── verify remote host
            ├── verify /srv/backups mount
            ├── verify target write access
            └── rsync mirror
                    │
                    ▼
              Raspberry Pi 5
              <BACKUP_HOSTNAME>
              <BACKUP_SERVER_IP>
                    │
                    └── 4 TB NVMe SSD
                        └── /srv/backups/palworld
```

This setup provides a lightweight, largely self-maintaining Palworld server with automatic game updates, controlled restarts, Debian package maintenance, health checks, built-in world backups, and a 30-minute off-host backup mirror to a separate physical machine and storage device.
