# Dart Services Compose

**English** | [한국어](#한국어)

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
| `SDK_VERSION` | `sdk7` | SDK version (`sdk2` ~ `sdk7`) |
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
└── sdk7/
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

**Fix:** always run from the same folder, or pin the project name so it stays the same everywhere (`dart-services-compose` is what a plain `git clone` already uses):

```shell
docker compose -p dart-services-compose --profile build up -d
```

If a container is genuinely orphaned, remove it:

```shell
docker rm -f simulator
```

#### The first `docker compose up` takes several minutes

The first run downloads about 800 MB of images for the default set (they expand to several GB on disk). It is not stuck. Later starts take seconds.

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

The `simulator` and `build-module-fw` images are `linux/amd64` only, so on Apple Silicon they run under emulation. A noticeably slow first start is expected.

Do **not** enable Rancher Desktop's Rosetta support (Preferences → Virtual Machine → Emulation). It is not supported here — the simulator fails to start with it enabled.

### Windows

#### Rancher Desktop does not start at all (`wsl.exe exited with code 4294967295`, `HCS_E_CONNECTION_TIMEOUT`)

First check whether WSL itself is healthy:

```powershell
wsl -d Ubuntu -- echo ok
```

(Replace `Ubuntu` with any other distribution you have — `wsl --list` shows them. If you have none, skip this check.)

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

---

# 한국어

[English](#dart-services-compose) | **한국어**

Dart 로봇 시뮬레이터와 빌드 서비스를 실행하기 위한 Docker Compose 구성입니다.

`docker compose`를 지원하는 환경이라면 어디서든 사용할 수 있습니다 (예: Docker Desktop, Rancher Desktop, OrbStack, Podman Desktop).

## 사전 준비

**Git**
- Windows: `winget install Git.Git` (또는 [git-scm.com](https://git-scm.com/)에서 직접 다운로드)
- Mac: `brew install git`

> Git은 필수가 아닙니다. GitHub에서 저장소를 ZIP으로 내려받아(**Code ▸ Download ZIP**) 빠른 시작의 1번 단계를 건너뛰어도 됩니다.

**컨테이너 런타임** (아래 중 하나를 설치):

**Docker Desktop** (상업적 사용 시 유료)

- Windows: `winget install Docker.DockerDesktop`
- Mac: `brew install --cask docker-desktop`

**Rancher Desktop** (무료)

- Windows: `winget install suse.RancherDesktop`
- Mac: `brew install --cask rancher`

> **Rancher Desktop만 해당:** 설치 후 컨테이너 엔진을 반드시 **dockerd (moby)**로 선택해야 합니다 (Preferences → Container Engine). `containerd` 엔진에는 Compose 플러그인이 포함되어 있지 않아 아래의 모든 `docker compose` 명령이 실패합니다. → [가이드](https://docs.rancherdesktop.io/ui/preferences/container-engine/general)

> **Mac + Rancher Desktop:** Docker 소켓 경로가 기본값과 다릅니다.
> `~/.zshrc`에 아래 내용을 추가하면 영구적으로 적용됩니다:
> ```shell
> export DOCKER_HOST=unix://$HOME/.rd/docker.sock
> ```

> **Mac + Rancher Desktop:** 기본 VM 사양은 2 CPU / 4 GB로, `.env.example`의 기본값인 `SIMULATOR_CPU=4`보다 작습니다. 처음 실행하기 전에 Preferences (⌘,) → Virtual Machine → Hardware에서 **CPU 4개 이상 / 메모리 6 GB 이상**으로 올리세요. 그렇지 않으면 시뮬레이터가 시작되지 않습니다. [macOS 전용](#macos-전용) 항목을 참고하세요.

## 빠른 시작

```shell
# 1. 클론
git clone https://github.com/DoosanRobotics/dart-services-compose.git
cd dart-services-compose

# 2. 환경 설정
cp .env.example .env

# 3. 시뮬레이터 + 빌드 모듈 시작
# 참고: Docker Desktop 또는 Rancher Desktop 이 먼저 실행 중이어야 합니다
docker compose --profile build up -d
```

## 설정

`.env` 파일을 수정해 값을 변경할 수 있습니다:

| 변수 | 기본값 | 설명 |
|---|---|---|
| `SDK_VERSION` | `sdk5` | SDK 버전 (`sdk2` ~ `sdk5`) |
| `ROBOT_MODEL` | `M1013` | 로봇 모델 (아래 목록 참고) |
| `SIMULATOR_CPU` | `4` | 시뮬레이터에 할당할 CPU 코어 수 |
| `SIMULATOR_MEMORY` | `2g` | 시뮬레이터에 할당할 메모리 |

### 로봇 모델

**sdk2 이상:** M0609, M0617, M1013, M1509, A0509, A0912, H2017, H2515, E0509

**sdk4 이상:** P3020

## 사용법

```shell
# 시뮬레이터 + 빌드 모듈 시작
docker compose --profile build up -d

# 시뮬레이터만 시작:
#docker compose up -d

# 중지
docker compose --profile build down
```

시뮬레이터 데이터는 `docker compose up`을 실행한 디렉터리 아래에 자동으로 생성됩니다:

```
data/
└── sdk5/
    └── M1013/
```

## 업데이트

```shell
docker compose --profile build pull
docker compose --profile build up -d
```

## 이미지

이미지는 GitHub Container Registry (GHCR)에서 제공되며, `docker compose up` 시 자동으로 내려받습니다.

- [`ghcr.io/doosanrobotics/simulator`](https://github.com/orgs/DoosanRobotics/packages/container/package/simulator)
- [`ghcr.io/doosanrobotics/build-module-fw`](https://github.com/orgs/DoosanRobotics/packages/container/package/build-module-fw)
- [`ghcr.io/doosanrobotics/build-module-ui`](https://github.com/orgs/DoosanRobotics/packages/container/package/build-module-ui)

## 문제 해결

바로가기: [공통](#공통) · [macOS 전용](#macos-전용) · [Windows 전용](#windows-전용)

### 공통

#### `docker: 'compose' is not a docker command` 또는 `unknown flag: --profile`

두 오류의 원인은 같습니다. 컨테이너 엔진이 `containerd`로 설정되어 있기 때문입니다. 이 모드의 `docker`는 Compose 플러그인이 빠진 nerdctl 래퍼라 Compose 명령을 인식하지 못합니다.

**해결:** Rancher Desktop → Preferences → Container Engine → **dockerd (moby)** → Apply (약 1분 소요). **새 터미널**을 열고 확인합니다:

```shell
docker compose version
```

#### `Cannot connect to the Docker daemon`

컨테이너 런타임에 연결할 수 없는 상태입니다.

- Docker Desktop 또는 Rancher Desktop이 실제로 실행 중인지 확인하세요 (Windows는 작업 표시줄 트레이, Mac은 메뉴 막대). 방금 실행했다면 1분 정도 기다리세요.
- Mac + Rancher Desktop: `DOCKER_HOST`가 설정되어 있어야 합니다 ([사전 준비](#사전-준비) 참고). `echo $DOCKER_HOST` 실행 시 `unix:///Users/<사용자>/.rd/docker.sock`이 출력되어야 합니다.

#### `Conflict. The container name "/simulator" is already in use`

`docker-compose.yml`은 `container_name`을 고정값으로 지정하지만, Compose 프로젝트 이름은 명령을 실행한 디렉터리 이름을 따라 정해집니다. 서로 다른 두 폴더에서 `up`을 실행하면(예: ZIP으로 받은 `dart-services-compose-main`과 `git clone` 한 폴더) 이름이 충돌합니다.

**해결:** 항상 같은 폴더에서 실행하거나, 어디서 실행하든 프로젝트 이름을 같은 값으로 지정하세요 (`dart-services-compose`는 일반적인 `git clone`이 이미 사용하는 이름입니다):

```shell
docker compose -p dart-services-compose --profile build up -d
```

컨테이너가 정리되지 않고 남아 있다면 직접 삭제합니다:

```shell
docker rm -f simulator
```

#### 첫 `docker compose up`이 몇 분씩 걸립니다

첫 실행에서는 기본 구성 기준으로 약 800 MB의 이미지를 내려받습니다 (압축 해제 후 디스크에서는 수 GB를 차지합니다). 멈춘 것이 아닙니다. 이후 실행은 수 초면 끝납니다.

#### `docker compose logs`에 아무것도 출력되지 않습니다

`docker-compose.yml`의 세 서비스 모두 `logging: driver: none`으로 선언되어 있어 Compose가 읽을 로그가 없습니다. 출력이 비어 있거나 로깅 드라이버가 읽기를 지원하지 않는다는 메시지가 나오는 것은 정상이며, 컨테이너에 문제가 생겼다는 뜻이 아닙니다.

시뮬레이터 로그는 컨테이너 내부에 기록됩니다. 아래 명령으로 호스트에 복사할 수 있습니다:

```shell
# Mac
mkdir -p ~/logs && docker cp simulator:/home/dra/etc/logs/dart-suite ~/logs
```

```powershell
# Windows PowerShell
mkdir C:\logs -Force; docker cp simulator:/home/dra/etc/logs/dart-suite C:\logs
```

컨테이너가 한 번이라도 시작된 적이 있다면 실행 중이든 중지 상태든 동작합니다. 버그를 신고할 때 이 폴더를 첨부해 주세요.

#### `.env`가 무시됩니다 (SDK 버전이나 로봇 모델이 의도와 다르게 적용되거나, `data/`가 엉뚱한 위치에 생성됨)

Compose는 명령을 실행한 디렉터리에서 `.env`를 읽습니다. 다른 위치에서 실행하면 아무 경고 없이 기본값으로 동작하고, `data/`도 그 위치에 생성됩니다.

**해결:** 항상 `.env`가 있는 폴더에서 `docker compose`를 실행하세요. `ls -a` (Mac) 또는 `dir` (Windows)로 `.env`가 보이는지 확인할 수 있습니다.

#### `Memory swappiness discarded` 경고

`docker-compose.yml`이 `mem_swappiness`를 지정하지만 Rancher Desktop / WSL2 커널이 이 값을 무시하면서 나오는 알림입니다. 오류가 아니며 별도 조치가 필요 없습니다.

### macOS 전용

#### `range of CPUs is from 0.01 to 2.00, as there are only 2 CPUs available`

`.env`에는 `SIMULATOR_CPU=4`로 지정되어 있지만, Rancher Desktop VM의 기본값은 2 CPU / 4 GB입니다.

**해결 (권장):** Preferences (⌘,) → **Virtual Machine** → **Hardware**에서 **CPUs**를 4개 이상, **Memory**를 6 GB 이상으로 설정하고 Apply를 누릅니다. VM이 재시작됩니다 (약 30초).

**대안:** `.env`의 `SIMULATOR_CPU` 값을 VM에 할당된 코어 수 이하로 낮춥니다.

#### 호스트에서 UDP 포트가 열리지 않습니다

`docker-compose.yml`은 `12360/udp` (시뮬레이터 자동 검색)와 `8089/udp` (build-module-ui)를 노출하지만, 호스트에서는 이 포트에 접근할 수 없습니다. Rancher Desktop의 기본 포트 포워더(SSH)가 **TCP 만** 전달하기 때문입니다.

**해결:** gRPC 포워더로 전환합니다. 설치 후 한 번만 실행하면 됩니다:

```shell
rdctl set --experimental.virtual-machine.ssh-port-forwarder=false
```

Rancher Desktop이 자동으로 재시작됩니다 (약 30초). 포트가 바인딩되었는지 확인합니다:

```shell
lsof -iUDP:12360
```

Rancher Desktop을 재설치하거나 `rdctl factory-reset`을 실행한 뒤에는 다시 설정해야 합니다. Windows에서는 필요하지 않습니다.

#### Apple Silicon: 첫 실행이 느리며, Rosetta는 반드시 꺼 두어야 합니다

`simulator`와 `build-module-fw` 이미지는 `linux/amd64` 전용이므로 Apple Silicon에서는 에뮬레이션으로 실행됩니다. 첫 실행이 눈에 띄게 느린 것은 정상입니다.

Rancher Desktop의 Rosetta 지원(Preferences → Virtual Machine → Emulation)은 **켜지 마세요.** 지원되지 않으며, 켜면 시뮬레이터가 시작되지 않습니다.

### Windows 전용

#### Rancher Desktop이 아예 실행되지 않습니다 (`wsl.exe exited with code 4294967295`, `HCS_E_CONNECTION_TIMEOUT`)

먼저 WSL 자체가 정상인지 확인합니다:

```powershell
wsl -d Ubuntu -- echo ok
```

(`Ubuntu`는 설치된 다른 배포판 이름으로 바꿔도 됩니다. `wsl --list`로 목록을 확인할 수 있으며, 배포판이 하나도 없다면 이 확인은 건너뛰세요.)

다른 배포판이 정상적으로 응답한다면 WSL은 문제가 없고 Rancher Desktop 전용 배포판만 손상된 것입니다. 트레이에서 Rancher Desktop을 완전히 종료한 뒤 **관리자 권한 PowerShell**에서 실행합니다:

```powershell
rdctl factory-reset
wsl --shutdown

# 아래 목록에 실제로 표시되는 배포판만 unregister 하세요
wsl --list --verbose
wsl --unregister rancher-desktop
wsl --unregister rancher-desktop-data

Remove-Item -Recurse -Force "$env:LOCALAPPDATA\rancher-desktop" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:APPDATA\rancher-desktop" -ErrorAction SilentlyContinue
```

그 뒤 Rancher Desktop을 제거하고 다시 설치하면 첫 실행 시 배포판이 새로 생성됩니다.

#### `WSL2 installation is incomplete`

WSL 코어가 없거나 오래된 상태입니다:

```powershell
wsl --update
```

이후 Rancher Desktop을 다시 실행하세요.

#### 재부팅해도 컨테이너 엔진이 끝까지 시작되지 않습니다

펌웨어에서 하드웨어 가상화가 비활성화되어 있습니다.

- BIOS/UEFI → Advanced (또는 Security ▸ CPU Setup) → **Virtualization** / **Intel(R) Virtualization Technology** → Enable → F10으로 저장.
- 또는 Windows에서: 설정 ▸ 업데이트 및 보안 ▸ 복구 ▸ **지금 다시 시작** → 문제 해결 ▸ 고급 옵션 ▸ **UEFI 펌웨어 설정**.

회사에서 관리하는 PC는 이 설정이 잠겨 있을 수 있습니다. IT 담당자에게 문의하세요.

#### Rancher Desktop 진단(Diagnostics) 탭에 오류가 표시됩니다

Docker 연결 오류가 발생하거나, Rancher Desktop Diagnostics 탭에 `Docker context is currently desktop-linux instead of default` 또는 `C:\Windows\system32\wsl.exe exited with code 1` 같은 문제가 표시되면 PowerShell에서 아래 방법을 시도하세요:

1. **Docker Desktop과 Rancher Desktop 충돌**
   Windows에서 두 프로그램을 동시에 실행하면 Docker CLI 혼선, WSL 충돌, 포트 바인딩 오류가 발생할 수 있습니다.

   **해결**: **Docker Desktop 제거**를 강력히 권장합니다. 둘 다 유지해야 한다면 반드시 *하나만* 실행한 상태로 두고, CLI가 올바른 엔진을 가리키도록 되돌리세요:
   ```powershell
   docker context use default
   ```

2. **중지된 WSL 배포판 깨우기**
   기본 WSL 배포판(예: `Ubuntu`)이 중지되어 있으면 Rancher Desktop이 백그라운드 구성 요소를 배치하지 못할 수 있습니다. 아래 명령으로 직접 깨웁니다:
   ```powershell
   wsl -d Ubuntu -e true
   ```
   실행 후 Rancher Desktop Diagnostics 탭을 다시 확인하거나 Rancher Desktop을 재시작하세요.

3. **`wsl.exe exited with code 1` 해결 (kubeconfig / WSL 오류)**
   Rancher Desktop Diagnostics에 `Error managing distribution Ubuntu: kubeconfig: C:\Windows\system32\wsl.exe exited with code 1` 같은 오류가 표시된다면, 충돌하는 `~/.kube/config` 파일 또는 오래된 WSL 코어가 원인일 수 있습니다.

   **해결 A: `kubeconfig` 파일 충돌 (invalid argument)**
   Rancher Desktop은 WSL 배포판(예: `Ubuntu`) 안의 `~/.kube/config`에 자체 클러스터 설정을 가리키는 심볼릭 링크를 만들려고 합니다. 해당 경로에 일반 파일이나 디렉터리가 이미 있으면 부팅 과정이 실패합니다. 백업하거나 이동해서 해결합니다:
   ```powershell
   wsl -d Ubuntu -- mv ~/.kube/config ~/.kube/config.bak
   ```
   *(참고: `Ubuntu`는 기본 WSL 배포판 이름이 다르다면 그에 맞게 바꾸세요.)*

   **해결 B: 오래되었거나 멈춘 WSL 코어**
   문제가 계속되면 WSL 코어가 오래되었거나 멈춘 상태일 수 있습니다. 업데이트 후 강제로 재시작합니다:
   ```powershell
   wsl --update
   wsl --shutdown
   ```
   어느 방법을 적용하든 마지막에 Rancher Desktop을 완전히 종료했다가 다시 실행하세요.

#### 오래 켜 둘수록 Windows가 느려집니다

WSL2 VM이 한 번 확보한 메모리를 반환하지 않기 때문입니다. `%USERPROFILE%\.wslconfig`에서 상한을 지정하세요:

```ini
[wsl2]
processors=4
memory=6GB
```

그런 다음 한 번 적용합니다:

```powershell
wsl --shutdown
```

#### 호스트 포트가 사용 중이라 컨테이너가 시작되지 않습니다

Windows에서는 다른 서비스나 이전 Docker 세션이 필요한 호스트 포트를 점유하고 있어 컨테이너가 바인딩하지 못할 수 있습니다. 서비스별 사용 포트는 다음과 같습니다:

| 서비스 | 컨테이너 이름 | 호스트 포트 |
|---|---|---|
| `simulator` | `simulator` | 12345, 3601, 502, 12360 (UDP) |
| `build_fw` | `build-module-fw` | 5022 |
| `build_ui` | `build-module-ui` | 5023, 3002, 8089 (UDP) |

**1단계 — 어떤 포트가 사용 중인지 확인:**

```powershell
# 필요한 포트가 이미 점유되어 있는지 확인
netstat -ano | Select-String ":5023\b|:3002\b|:8089\b|:5022\b|:12345\b|:12360\b|:3601\b|:502\b"
```

마지막 열의 PID를 확인한 뒤 해당 프로세스를 조회합니다:

```powershell
Get-Process -Id <PID> -ErrorAction SilentlyContinue
```

아무것도 출력되지 않고 PID가 `0` 또는 `4` 라면, 해당 포트는 종료할 수 있는 애플리케이션이 아니라 Windows 커널(`System` / `System Idle Process`)이 예약한 것입니다. 2단계로 넘어가거나 재부팅해 예약을 해제하세요.

**2단계 — 남아 있는 WSL2 포트 프록시 규칙 확인** (**관리자 권한**으로 PowerShell 실행):

```powershell
netsh interface portproxy show all
```

컨테이너가 실행 중이 아닌데도 위 포트들이 WSL2 IP(예: `172.x.x.x`)를 가리키는 항목이 보인다면, 이전 실행에서 정리되지 않고 남은 규칙입니다. 삭제하세요 (이 역시 관리자 권한 필요):

```powershell
# 특정 규칙 삭제 (<PORT> 를 충돌하는 포트 번호로 바꾸세요)
netsh interface portproxy delete v4tov4 listenport=<PORT> listenaddress=0.0.0.0
```

**3단계 — 컨테이너 재시작:**

```powershell
docker compose --profile build down
docker compose --profile build up -d
```

> **팁:** Windows 서비스(예: `svchost / iphlpsvc`)가 포트를 잡고 있다면 Docker를 재시작하거나 PC를 재부팅해 예약을 해제해 보세요.

## 라이선스

Copyright © Doosan Robotics. All rights reserved.

이 저장소는 개인적·비상업적 용도로만 제공됩니다.
두산로보틱스의 사전 서면 동의 없이 이 저장소의 어떤 부분도 수정, 재배포하거나 상업적으로 이용할 수 없습니다.

> 이 번역은 편의를 위한 참고용이며, 법적 효력은 영문 원문에 있습니다. 두 문안이 다를 경우 영문이 우선합니다.
>
> *This Korean text is a convenience translation. The English version above is the legally binding one and prevails in case of any discrepancy.*
