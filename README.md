# Dart Services Compose

Docker Compose setup for running Dart robot simulator and build services.

This setup works with any environment that supports `docker compose` (e.g. Docker Desktop, Rancher Desktop, OrbStack, Podman Desktop).

## Prerequisites

Install one of the following:

**Docker Desktop** (paid for commercial use)

- Windows: `winget install Docker.DockerDesktop`
- Mac: `brew install --cask docker`

**Rancher Desktop** (free)

- Windows: `winget install suse.RancherDesktop`
- Mac: `brew install --cask rancher`
  - Select **dockerd (moby)** as the container runtime after installation → [Guide](https://docs.rancherdesktop.io/ui/preferences/container-engine/general)

> **Mac + Rancher Desktop:** The Docker socket path differs from the default.
> Add the following to `~/.zshrc` to apply permanently:
> ```shell
> export DOCKER_HOST=unix://$HOME/.rd/docker.sock
> ```

## Quick Start

```shell
# 1. Clone this repository
git clone https://github.com/dsg-robotics/dart-services-compose.git
cd dart-services-compose

# 2. Configure environment
cp .env.example .env

# 3. Start simulator
docker compose up -d
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
# Option 1: Start simulator only
docker compose up -d

# Option 2: Start simulator + build modules
# Remove '#' to enable
#docker compose --profile build up -d

# Stop
docker compose down
```

Simulator data is automatically created under the directory where you run `docker compose up`:

```
data/
└── sdk5/
    └── M1013/
```

## Update

```shell
docker compose pull
docker compose up -d
```

## Services

### Simulator (default)

| Port | Protocol | Description |
|---|---|---|
| 12345 | TCP | DRCF |
| 3601 | TCP | DRAS |
| 502 | TCP | Modbus |
| 12360 | UDP | — |

### Build Modules (optional)

Start with the `build` profile:

```shell
docker compose --profile build up -d
```

| Service | Port | Description |
|---|---|---|
| build-module-fw | 5022 | Firmware builder (SSH) |
| build-module-ui | 3002 | UI build service |
| build-module-ui | 5023 | UI builder (SSH) |

## Images

- [`ghcr.io/dsg-robotics/simulator`](https://github.com/orgs/dsg-robotics/packages/container/package/simulator)
- [`ghcr.io/dsg-robotics/build-module-fw`](https://github.com/orgs/dsg-robotics/packages/container/package/build-module-fw)
- [`ghcr.io/dsg-robotics/build-module-ui`](https://github.com/orgs/dsg-robotics/packages/container/package/build-module-ui)

## License

Copyright © Doosan Robotics. All rights reserved.

This repository is provided for personal, non-commercial use only.
Modification, redistribution, or commercial use of any part of this repository is not permitted without prior written consent from Doosan Robotics.
