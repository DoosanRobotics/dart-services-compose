# Dart Services Compose

Docker Compose setup for running Dart robot simulator and build services.

This setup works with any environment that supports `docker compose` (e.g. Docker Desktop, Rancher Desktop, OrbStack, Podman Desktop).

## Prerequisites

**Git**
- Windows: `winget install Git.Git` (or download from [git-scm.com](https://git-scm.com/))
- Mac: `brew install git`

**Container Runtime** (Install one of the following):

**Docker Desktop** (paid for commercial use)

- Windows: `winget install Docker.DockerDesktop`
- Mac: `brew install --cask docker`

**Rancher Desktop** (free)

- Windows: `winget install suse.RancherDesktop`
- Mac: `brew install --cask rancher`

> **Important:** You MUST select **dockerd (moby)** as the container runtime after installation → [Guide](https://docs.rancherdesktop.io/ui/preferences/container-engine/general)

> **Mac + Rancher Desktop:** The Docker socket path differs from the default.
> Add the following to `~/.zshrc` to apply permanently:
> ```shell
> export DOCKER_HOST=unix://$HOME/.rd/docker.sock
> ```

## Quick Start

```shell
# 1. Clone
git clone https://github.com/DoosanRobotics/dart-services-compose.git
cd dart-services-compose

# 2. Configure environment
cp .env.example .env

# 3. Start simulator + build modules
# Note: Make sure Docker Desktop or Rancher Desktop is running first
docker compose --profile build up -d
```

## Configuration

Edit `.env` to customize:

| Variable | Default | Description |
|---|---|---|
| `SDK_VERSION` | `sdk5` | SDK version (`sdk2` ~ `sdk5`) |
| `ROBOT_MODEL` | `M1013` | Robot model (see options below) |
| `SIMULATOR_CPU` | `4` | CPU cores allocated to simulator |
| `SIMULATOR_MEMORY` | `2g` | Memory allocated to simulator |

### Robot Models

**sdk2 and above:** M0609, M0617, M1013, M1509, A0509, A0912, H2017, H2515, E0509

**sdk4 and above:** P3020

## Usage

```shell
# Start simulator + build modules
docker compose --profile build up -d

# Start simulator only:
#docker compose up -d

# Stop
docker compose --profile build down
```

Simulator data is automatically created under the directory where you run `docker compose up`:

```
data/
└── sdk5/
    └── M1013/
```

## Update

```shell
docker compose --profile build pull
docker compose --profile build up -d
```

## Images

Images are hosted on GitHub Container Registry (GHCR) and pulled automatically on `docker compose up`.

- [`ghcr.io/doosanrobotics/simulator`](https://github.com/orgs/DoosanRobotics/packages/container/package/simulator)
- [`ghcr.io/doosanrobotics/build-module-fw`](https://github.com/orgs/DoosanRobotics/packages/container/package/build-module-fw)
- [`ghcr.io/doosanrobotics/build-module-ui`](https://github.com/orgs/DoosanRobotics/packages/container/package/build-module-ui)

## Troubleshooting

### Rancher Desktop (Windows)

If you encounter Docker connectivity errors or if the Rancher Desktop Diagnostics tab reports problems such as `Docker context is currently desktop-linux instead of default` or `C:\Windows\system32\wsl.exe exited with code 1`, try the following fixes in your terminal (PowerShell):

1. **Conflicts between Docker Desktop and Rancher Desktop**
   Running both simultaneously on Windows can cause Docker CLI confusion, WSL collisions, and port binding errors.
   
   **Fix**: We highly recommend **uninstalling Docker Desktop**. If you must keep both, ensure only *one* is running at a time and run `docker context use default`.

2. **Set Default Docker Context**
   Ensure Docker is pointing to the correct engine:
   ```shell
   docker context use default
   ```

3. **Wake Up the Stopped WSL Distribution**
   If your default WSL distribution (e.g., `Ubuntu`) is stopped, Rancher Desktop may fail to inject its background helpers. Manually wake it up by running:
   ```shell
   wsl -d Ubuntu -e true
   ```
   After running this, check the Rancher Desktop Diagnostics tab again or restart Rancher Desktop.

4. **Resolve `wsl.exe exited with code 1` (kubeconfig / WSL Error)**
   If Rancher Desktop Diagnostics reports an error like `Error managing distribution Ubuntu: kubeconfig: C:\Windows\system32\wsl.exe exited with code 1`, this may be caused by a conflicting `~/.kube/config` file or an outdated WSL core.

   **Fix A: Conflicting `kubeconfig` file (invalid argument)**
   Rancher Desktop attempts to create a symlink at `~/.kube/config` in your WSL distribution (e.g., `Ubuntu`) to point to its own cluster configuration. If a regular file or directory already exists at this path, the boot process will fail. Fix this by backing up or removing the file:
   ```shell
   wsl -d Ubuntu -- mv ~/.kube/config ~/.kube/config.bak
   ```
   *(Note: Replace `Ubuntu` with your default WSL distribution name if it differs.)*

   **Fix B: Outdated or stuck WSL core**
   If the issue persists, your WSL core might be outdated or stuck. Fix this by updating and forcefully restarting WSL:
   ```shell
   wsl --update
   wsl --shutdown
   ```
   After applying either fix, completely quit and relaunch Rancher Desktop.

### Container Fails to Start Due to Windows Port Conflicts

On Windows, services or previous Docker sessions may occupy required host ports, preventing containers from binding to them. Affected ports per service:

| Service | Host Ports |
|---|---|
| `simulator` | 1122, 12345, 12360 (UDP), 3601, 502 |
| `build-module-fw` | 5022 |
| `build-module-ui` | 5023, 3002, 8089 (UDP) |

**Step 1 — Identify which ports are in use:**

```powershell
# Check if any required ports are already occupied
netstat -ano | Select-String "5023|3002|8089|5022|1122|12345|12360|3601|:502 "
```

Note the PID from the last column and identify the process:

```powershell
Get-Process -Id <PID>
```

**Step 2 — Check for stale WSL2 port proxy rules:**

```powershell
netsh interface portproxy show all
```

If you see entries for the above ports pointing to a WSL2 IP (e.g., `172.x.x.x`) but containers are not running, these are stale rules. Remove them:

```shell
# Remove a specific stale rule (replace <PORT> with the conflicting port number)
netsh interface portproxy delete v4tov4 listenport=<PORT> listenaddress=0.0.0.0
```

**Step 3 — Restart containers:**

```shell
docker compose --profile build down
docker compose --profile build up -d
```

> **Tip:** If a Windows service (e.g., `svchost / iphlpsvc`) is holding the port, try restarting Docker or rebooting the machine to release the reservation.

## License

Copyright © Doosan Robotics. All rights reserved.

This repository is provided for personal, non-commercial use only.
Modification, redistribution, or commercial use of any part of this repository is not permitted without prior written consent from Doosan Robotics.
