# home-server-setup
Collection of Setup Files for Smart Home Servers & Satellites


## Overview
The initial set of files assumes an Ubuntu 22.04 LTS (Jammy) or compatible system using the `apt` tool.
The scripts also require ansible and git. Feel free to fork and/or submit pull requests for other systems!

All playbooks include the `server_core` stack, which includes the following services:
- **Portainer**: Web UI container management (via docker / podman)
- **Syncthing**: File syncing and versioning (for backups / rollback)

These playbooks are available in this repo:
| Playbook                           | Description
|--                                  |--
| **`rhasspy-sattelite-setup.yml`**  | Rhasspy Sattelite (room listener) setup
| **`homeassistant-base-setup.yml`** | Home Assistant core with Mosquitto (MQTT), Rhasspy Base Station, and Z-Wave JS Server


## Preparation
To run a playbook on a fresh Jammy install, use the following commands:

1. `sudo apt update`
2. `sudo apt install ansible git`


## Deployment
Use `ansible-pull` to install the desired system. For example, to set up a Rhasspy sattelite listener, issue
this command:

- `sudo ansible-pull -U https://github.com/doctorjei/home-server-setup.git -i localhost
rhasspy-sattelite-setup.yml`


## Connecting
Services are allocated at their default ports. Here are some of them:

| Service          | Port(s)                                                                           |||
|--                |--                                                                             |--|--|
| Home Assistant   | 8123/tcp (Web UI / Websocket)                                                     |||
| Mosquitto        | 1883/tcp (Event Traffic)                                                          |||
| Portainer        | 9443/tcp (Web UI) | 8000/tcp (Edge Point Access)                                   ||
| Rhasspy          | 12101/tcp (Web UI)                                                                |||
| Syncthing        | 8384/tcp (Web UI) | 21027/udp (Discovery) | 22000/tcp/udp (Peer Connect / Transfer) |
| Z-Wave JS Server | 8901/tcp (Web UI) | 3000/tcp (Websocket)                                           ||
