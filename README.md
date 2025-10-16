# twingate-rootless-updater
> Auto-update a **Twingate Connector** running under **rootless Docker** in two ways: **systemd script** or **container**.  
> The vendor script is patched to remove a root-only `sysctl` that breaks rootless Docker.

## Contents
- [Overview](#overview)
- [Why this exists](#why-this-exists)
- [How to use](#how-to-use)
- [Requirements](#requirements)
- [Option 1: install systemd script (rootless user service + timer)](#option-1-install-systemd-script-rootless-user-service--timer)
  - [To run](#to-run)
  - [Uninstall](#uninstall)
- [Option 2 — container (self-contained updater)](#option-2--container-selfcontained-updater)
  - [Build run](#to-run)
  - [docker compose example](#docker-compose-example)
  - [Uninstall](#uninstall-1)
- [Troubleshooting](#troubleshooting)
  - [Change update frequency](#change-update-frequency)
- [Security notes](#security-notes)
- [FAQ](#faq)
- [License](#license)


## Overview
This repo provides two (hopefully) deployment-friendly ways to keep an **updated Twingate Connector** running under **rootless Docker** automatically:

- **systemd script**:

 Installs a **user** service & timer that runs Twingate’s official `docker-upgrade.sh` against your **rootless** Docker socket, with a small patch applied. Best for local and managed installs ✔️
- **container**

Runs a lightweight updater container that periodically fetches the vendor script, patches it, and executes it against your **rootless** Docker via a mounted socket. Immediate setup & forget ✔️

Both will:
- preserve your existing container’s **env** and **name**
- remove the root-only sysctl flag
- and can run alongside any existing services

---

### Why this exists
Twingate’s upstream script recreates the container with:
```bash
--sysctl net.ipv4.ping_group_range="0 2147483647"
```
On many **rootless** Docker setups this causes the following error:
```
error during container init: write /proc/sys/net/ipv4/ping_group_range: invalid argument
```
We strip away *just* this flag. Nothing else changes with their upstream script.
## How to use:
```
twingate-rootless-updater/
├─ README.md
├─ systemd-update-script/
│ ├─ install-twingate-rootless-updater.sh
│ ├─ systemd/
│ │ ├─ twingate-upgrade.service
│ │ └─ twingate-upgrade.timer
│ └─ scripts/
│ └─ tg-upgrade-patched
└─ update-container/
├─ Dockerfile
├─ entrypoint.sh
├─ docker-compose.example.yml
└─ test-run.sh
```
## Requirements
- Rootless Docker running for the **target user**
    - You can run ``systemctl --user status docker`` to confirm
- A running Twingate Connector container created by that user (the updater discovers it automatically)
- Linux with systemd (only if using option 1)


## Option 1: install systemd script (rootless user service + timer)


Run the installer script with **sudo** (or as root), **also adding the targeted user** who owns the rootless Docker daemon:
```bash
git clone https://github.com/Mail222/twingate-rootless-updater.git
cd twingate-rootless-updater/systemd-update-script

# Example: install for user "bob"
sudo bash option-a-systemd/install-twingate-rootless-updater.sh bob
```
This creates our wrapper at ``~/.local/bin/tg-upgrade-patched`` as well as installing the user services to ``/etc/systemd/user`` and enables the service to run in the background/on startup.
- The **sudo/root** requirement is due to setting up under ``/etc/systemd/user`` and enabling on startup. Further releases may patch the need for sudo 🔄

---

### To run:
```bash
# If running in "target user" shell:
systemctl --user start twingate-upgrade.service

# If running as different user:
sudo -u <USER> systemctl --user start twingate-upgrade.service
```
### Uninstall:
```bash
sudo -u <USER> systemctl --user disable --now twingate-upgrade.timer
sudo rm -f /etc/systemd/user/twingate-upgrade.{service,timer}
sudo -u <USER> systemctl --user daemon-reload
sudo rm -f ~<USER>/.local/bin/tg-upgrade-patched
```


## Option 2: container (self-contained updater)
```bash
git clone https://github.com/Mail222/twingate-rootless-updater.git
cd twingate-rootless-updater/update-container
docker build -t twingate-updater:latest .
```
### To Run:
```bash
# Set the target username
TARGET_USER=username

UID_NUM=$(id -u $TARGET_USER)
GID_NUM=$(id -g $TARGET_USER)

docker run -d --name twingate-updater --restart unless-stopped \
  --user ${UID_NUM}:${GID_NUM} \
  -e SLEEP_SECONDS=86400 \
  -e DOCKER_HOST=unix:///sock/docker.sock \
  -v /run/user/${UID_NUM}/docker.sock:/sock/docker.sock \
  -v /home/<USER>/twingate:/work \
  twingate-updater:latest
```
---

### Uninstall
```bash
docker rm -f twingate-updater
```


## Troubleshooting