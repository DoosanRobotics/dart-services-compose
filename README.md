# Dart Services Compose

Docker Compose setup for running Dart robot simulator and build services.

This setup works with any environment that supports `docker compose` (e.g. Docker Desktop, Rancher Desktop, OrbStack, Podman Desktop).

## Prerequisites

**Git**
- Windows: `winget install Git.Git` (or download from [git-scm.com](https://git-scm.com/))
- Mac: `brew install git`

> Git is optional. You can also download the repository as a ZIP from GitHub (**Code ▸ Download ZIP**) and skip step 1 of the Quick Start.

**Container Runtime** (Install one of the following):

**Docker Desktop** (paid for commercial use)

- Windows: `winget install Docker.DockerDesktop`
- Mac: `brew install --cask docker-desktop`

**Rancher Desktop** (free)

- Windows: `winget install suse.RancherDesktop`
- Mac: `brew install --cask rancher`

> **Rancher Desktop only:** After installation you MUST select **dockerd (moby)** as the container engine (Preferences → Container Engine). The `containerd` engine does not ship the Compose plugin, so every `docker compose` command below will fail. → [Guide](https://docs.rancherdesktop.io/ui/preferences/container-engine/general)

> **Mac + Rancher Desktop:** The Docker socket path differs from the default.
> Add the following to `~/.zshrc` to apply permanently:
> ```shell
> export DOCKER_HOST=unix://$HOME/.rd/docker.sock
> ```

> **Mac + Rancher Desktop:** The default VM has 2 CPUs / 4 GB, which is less than the `SIMULATOR_CPU=4` default in `.env.example`. Raise it to **4+ CPUs / 6+ GB** in Preferences (⌘,) → Virtual Machine → Hardware before the first start, or the simulator will not start. See [macOS](#macos).

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

Jump to: [General](#general) · [macOS](#macos) · [Windows](#windows)

### General

#### `docker: 'compose' is not a docker command` or `unknown flag: --profile`

Both errors have the same cause: the container engine is set to `containerd`, whose `docker` command is a nerdctl shim without the Compose plugin.

**Fix:** Rancher Desktop → Preferences → Container Engine → **dockerd (moby)** → Apply (takes about a minute). Open a **new terminal**, then verify:

```shell
docker compose version
```

#### `Cannot connect to the Docker daemon`

The container runtime is not reachable.

- Check that Docker Desktop or Rancher Desktop is actually running (system tray on Windows, menu bar on Mac). If you just started it, give it about a minute.
- Mac + Rancher Desktop: `DOCKER_HOST` must be set (see [Prerequisites](#prerequisites)). `echo $DOCKER_HOST` should print `unix:///Users/<you>/.rd/docker.sock`.

#### `Conflict. The container name "/simulator" is already in use`

`docker-compose.yml` pins fixed `container_name` values, but the Compose project name is derived from the directory you run the command in. Running `up` from two different folders (for example an unzipped `dart-services-compose-main` and a `git clone`) creates the conflict.

**Fix:** always run from the same folder, or pin the project name to the same value every time:

```shell
docker compose -p dart-services --profile build up -d
```

If a container is genuinely orphaned, remove it:

```shell
docker rm -f simulator
```

#### The first `docker compose up` takes several minutes

The first run downloads roughly 3 GB of images. It is not stuck. Later starts take seconds.

#### `docker compose logs` shows nothing

All three services declare `logging: driver: none` in `docker-compose.yml`, so Compose has nothing to read. Empty output (or a message that the configured logging driver does not support reading) is expected and does not mean the containers failed.

Simulator logs are written inside the container. Copy them out:

```shell
# Mac
mkdir -p ~/logs && docker cp simulator:/home/dra/etc/logs/dart-suite ~/logs
```

```powershell
# Windows PowerShell
mkdir C:\logs -Force; docker cp simulator:/home/dra/etc/logs/dart-suite C:\logs
```

This works whether the container is running or stopped, as long as it has started at least once. Attach this folder when reporting a bug.

#### `.env` is ignored (wrong SDK version or robot model, `data/` in the wrong place)

Compose reads `.env` from the directory the command runs in. Started from anywhere else, it silently falls back to the built-in defaults and creates `data/` next to wherever you ran it.

**Fix:** always run `docker compose` from the folder that contains `.env`. Confirm with `ls -a` (Mac) or `dir` (Windows) — `.env` must be listed.

#### `Memory swappiness discarded` warning

`docker-compose.yml` sets `mem_swappiness`, which the Rancher Desktop / WSL2 kernel ignores. This is a notice, not an error. No action needed.

### macOS

#### `range of CPUs is from 0.01 to 2.00, as there are only 2 CPUs available`

`.env` requests `SIMULATOR_CPU=4`, but the Rancher Desktop VM defaults to 2 CPUs / 4 GB.

**Fix (recommended):** Preferences (⌘,) → **Virtual Machine** → **Hardware** → set **CPUs** to 4 or more and **Memory** to 6 GB or more → Apply. The VM restarts (about 30 seconds).

**Alternative:** lower `SIMULATOR_CPU` in `.env` to at most the number of cores the VM has.

#### UDP ports never appear on the host

`docker-compose.yml` publishes `12360/udp` (simulator discovery) and `8089/udp` (build-module-ui), but nothing on the host can reach them: Rancher Desktop's default port forwarder (SSH) forwards **TCP only**.

**Fix:** switch to the gRPC forwarder once per installation:

```shell
rdctl set --experimental.virtual-machine.ssh-port-forwarder=false
```

Rancher Desktop restarts automatically (about 30 seconds). Verify that the port is bound:

```shell
lsof -iUDP:12360
```

Repeat this after a Rancher Desktop reinstall or `rdctl factory-reset`. Not required on Windows.

#### Apple Silicon: slow first start, and Rosetta must stay off

The images are `linux/amd64` only, so on Apple Silicon they run under QEMU emulation. A noticeably slow first start is expected.

Do **not** enable Rancher Desktop's Rosetta support (Preferences → Virtual Machine → Emulation). It is not supported here — the simulator fails to start with it enabled.

### Windows

#### Rancher Desktop does not start at all (`wsl.exe exited with code 4294967295`, `HCS_E_CONNECTION_TIMEOUT`)

First check whether WSL itself is healthy:

```powershell
wsl -d Ubuntu -- echo ok
```

If another distribution answers normally, WSL is fine and only Rancher Desktop's own distributions are broken. Quit Rancher Desktop from the system tray, then run **PowerShell as Administrator**:

```powershell
rdctl factory-reset
wsl --shutdown

# Unregister only the distributions that actually appear in this list
wsl --list --verbose
wsl --unregister rancher-desktop
wsl --unregister rancher-desktop-data

Remove-Item -Recurse -Force "$env:LOCALAPPDATA\rancher-desktop" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:APPDATA\rancher-desktop" -ErrorAction SilentlyContinue
```

Then uninstall and reinstall Rancher Desktop. It recreates the distributions on first launch.

#### `WSL2 installation is incomplete`

The WSL core is missing or outdated:

```powershell
wsl --update
```

Restart Rancher Desktop afterwards.

#### The container engine never finishes starting, even after a reboot

Hardware virtualization is disabled in firmware.

- BIOS/UEFI → Advanced (or Security ▸ CPU Setup) → **Virtualization** / **Intel(R) Virtualization Technology** → Enable → F10 to save.
- Or from Windows: Settings ▸ Update & Security ▸ Recovery ▸ **Restart now** → Troubleshoot ▸ Advanced options ▸ **UEFI Firmware Settings**.

On a managed corporate PC this setting may be locked. Contact your IT administrator.

#### Rancher Desktop Diagnostics reports errors

If you encounter Docker connectivity errors, or if the Rancher Desktop Diagnostics tab reports problems such as `Docker context is currently desktop-linux instead of default` or `C:\Windows\system32\wsl.exe exited with code 1`, try the following fixes in PowerShell:

1. **Conflicts between Docker Desktop and Rancher Desktop**
   Running both simultaneously on Windows can cause Docker CLI confusion, WSL collisions, and port binding errors.

   **Fix**: We highly recommend **uninstalling Docker Desktop**. If you must keep both, ensure only *one* is running at a time and point the CLI back at the correct engine:
   ```powershell
   docker context use default
   ```

2. **Wake Up the Stopped WSL Distribution**
   If your default WSL distribution (e.g., `Ubuntu`) is stopped, Rancher Desktop may fail to inject its background helpers. Manually wake it up by running:
   ```powershell
   wsl -d Ubuntu -e true
   ```
   After running this, check the Rancher Desktop Diagnostics tab again or restart Rancher Desktop.

3. **Resolve `wsl.exe exited with code 1` (kubeconfig / WSL Error)**
   If Rancher Desktop Diagnostics reports an error like `Error managing distribution Ubuntu: kubeconfig: C:\Windows\system32\wsl.exe exited with code 1`, this may be caused by a conflicting `~/.kube/config` file or an outdated WSL core.

   **Fix A: Conflicting `kubeconfig` file (invalid argument)**
   Rancher Desktop attempts to create a symlink at `~/.kube/config` in your WSL distribution (e.g., `Ubuntu`) to point to its own cluster configuration. If a regular file or directory already exists at this path, the boot process will fail. Fix this by backing up or removing the file:
   ```powershell
   wsl -d Ubuntu -- mv ~/.kube/config ~/.kube/config.bak
   ```
   *(Note: Replace `Ubuntu` with your default WSL distribution name if it differs.)*

   **Fix B: Outdated or stuck WSL core**
   If the issue persists, your WSL core might be outdated or stuck. Fix this by updating and forcefully restarting WSL:
   ```powershell
   wsl --update
   wsl --shutdown
   ```
   After applying either fix, completely quit and relaunch Rancher Desktop.

#### Windows gets slower the longer the engine runs

The WSL2 VM keeps the memory it has claimed and does not return it. Cap it in `%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
processors=4
memory=6GB
```

Then apply the change once:

```powershell
wsl --shutdown
```

#### Containers fail to start because host ports are in use

On Windows, services or previous Docker sessions may occupy required host ports, preventing containers from binding to them. Affected ports per service:

| Service | Container name | Host Ports |
|---|---|---|
| `simulator` | `simulator` | 12345, 3601, 502, 12360 (UDP) |
| `build_fw` | `build-module-fw` | 5022 |
| `build_ui` | `build-module-ui` | 5023, 3002, 8089 (UDP) |

**Step 1 — Identify which ports are in use:**

```powershell
# Check if any required ports are already occupied
netstat -ano | Select-String ":5023\b|:3002\b|:8089\b|:5022\b|:12345\b|:12360\b|:3601\b|:502\b"
```

Note the PID from the last column and identify the process:

```powershell
Get-Process -Id <PID> -ErrorAction SilentlyContinue
```

If this returns nothing and the PID is `0` or `4`, the port is reserved by the Windows kernel (`System` / `System Idle Process`), not by an application you can close. Continue with Step 2, or reboot to release the reservation.

**Step 2 — Check for stale WSL2 port proxy rules** (run PowerShell **as Administrator**):

```powershell
netsh interface portproxy show all
```

If you see entries for the above ports pointing to a WSL2 IP (e.g., `172.x.x.x`) but containers are not running, these are stale rules. Remove them (also as Administrator):

```powershell
# Remove a specific stale rule (replace <PORT> with the conflicting port number)
netsh interface portproxy delete v4tov4 listenport=<PORT> listenaddress=0.0.0.0
```

**Step 3 — Restart containers:**

```powershell
docker compose --profile build down
docker compose --profile build up -d
```

> **Tip:** If a Windows service (e.g., `svchost / iphlpsvc`) is holding the port, try restarting Docker or rebooting the machine to release the reservation.

## License

Copyright © Doosan Robotics. All rights reserved.

This repository is provided for personal, non-commercial use only.
Modification, redistribution, or commercial use of any part of this repository is not permitted without prior written consent from Doosan Robotics.
