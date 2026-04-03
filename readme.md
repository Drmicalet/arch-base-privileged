# Arch Base Privileged

> A privileged Arch Linux base image for advanced container orchestration, AI/ML workloads, and systems requiring elevated permissions.

[![Docker Pull](https://img.shields.io/badge/pull-ghcr.io-blue)](https://ghcr.io/drmicalet/arch-base-privileged)
[![License](https://img.shields.io/badge/license-GPL--3.0-green)](LICENSE)

---

## Why This Image?

The official Arch Linux container images are minimal and excellent for many use cases, but they lack several features needed for serious container orchestration and development workflows. This image extends the official base with critical improvements while following Arch best practices.

| Feature | `archlinux:latest` | `archlinux:base-devel` | **arch-base-privileged** |
|---|---|---|---|
| DNS Resolution | May fail in builds | May fail in builds | **Improved with localhost fallback** |
| Privileged Mode | Not configured | Not configured | **Pre-configured** |
| AUR Helper (yay) | Not included | Not included | **Pre-installed with `builder` user** |
| Python Versions | 3.14 only | 3.14 only | **3.14 (default) + 3.12 via AUR** |
| Pacman Keyring | Manual setup | Manual setup | **Pre-initialized and trusted** |
| sudo (no password) | Not included | Not included | **Included** |
| systemd | Not included | Not included | **Enabled** |
| base-devel | Not included | Included | **Included** |
| Size | ~544 MB | ~915 MB | **~1.3 GB** |

---

## Key Features

### 1. Improved DNS Resolution

DNS resolution in containers can be unreliable, especially during image builds when `pacman -Syu` needs to fetch package databases. This image includes a custom resolver configuration with localhost fallback that eliminates "failed to synchronize databases" errors.

```dockerfile
# DNS works reliably during builds — no more sync failures
RUN pacman -Syu --noconfirm
```

### 2. AUR Helper (yay) with Builder User

AUR packages extend Arch with thousands of community-maintained packages. Since **yay cannot run as root** (by design), this image includes a dedicated `builder` user with sudo privileges, following Arch's official security best practice.

```dockerfile
# Install AUR packages safely — yay runs as non-root
RUN su - builder -c 'yay -S --noconfirm some-aur-package'
```

> **Important**: Always use `su - builder -c 'yay ...'` — never run yay as root.

### 3. Python 3.14 + Python 3.12 (Dual Version)

Arch Linux ships Python 3.14 as the system default, and that is what you get when you pull this image:

```
$ python --version
Python 3.14.3

$ python3 --version
Python 3.14.3

$ yay -S python312 --noconfirm   # Install 3.12 from AUR
$ python3.12 --version
Python 3.12.x
```

**Why this matters**: Many ML/AI frameworks, production libraries, and Python packages (like `pydantic-core` which requires Rust compilation) do not yet have pre-compiled wheels for Python 3.14. Python 3.12 has the most stable ecosystem with well-tested packages.

> **Read the [Python 3.12 Virtual Environment Guide](#python-312-virtual-environment-guide) below** for step-by-step instructions on creating a Python 3.12 venv using yay.

### 4. Pre-initialized Pacman Keyring

The Arch pacman keyring must be initialized and populated before installing signed packages. In standard images, this step must be done manually during each build, wasting time and sometimes failing with signature errors. This image ships with a fully initialized and trusted keyring.

```dockerfile
# Works immediately — no keyring setup needed
RUN pacman -S --noconfirm git curl base-devel
```

### 5. Localhost Registry Support

Built-in configuration for `localhost` container registries, essential for:

- Local development and testing workflows
- K3s / K3d clusters with local image registries
- Air-gapped environments with private registries

### 6. Privileged Container Ready

Pre-configured for privileged container operations, including:

- Hardware access (GPU, USB devices, /dev paths)
- System-level operations (mount, iptables, network config)
- K3s/Kubernetes privileged pods and DaemonSets

### 7. systemd Support

systemd is enabled and functional inside the container, allowing you to run services with full process management:

```dockerfile
docker run -d \
  --name arch-systemd \
  --privileged \
  --tmpfs /tmp \
  --tmpfs /run \
  --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  ghcr.io/drmicalet/arch-base-privileged:latest \
  /sbin/init
```

---

## Quick Start

### Pull the Image

```bash
# Podman (recommended)
podman pull ghcr.io/drmicalet/arch-base-privileged:latest

# Docker
docker pull ghcr.io/drmicalet/arch-base-privileged:latest
```

### Basic Usage

```bash
# Interactive shell
podman run -it --rm ghcr.io/drmicalet/arch-base-privileged:latest

# With privileged mode (hardware access)
podman run -it --rm --privileged ghcr.io/drmicalet/arch-base-privileged:latest

# Import into K3s/Kubernetes
podman save -o arch-base-privileged.tar ghcr.io/drmicalet/arch-base-privileged:latest
sudo k3s ctr images import arch-base-privileged.tar
```

### Example Dockerfile

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# System packages work immediately (DNS + keyring configured)
RUN pacman -Syu --noconfirm && \
    pacman -S --noconfirm git curl base-devel

# AUR packages with builder user (yay cannot run as root!)
RUN su - builder -c 'yay -S --noconfirm some-aur-package'

# Default python is 3.14
RUN python -m pip install fastapi uvicorn

WORKDIR /app
CMD ["/bin/bash"]
```

---

## Python 3.12 Virtual Environment Guide

Python 3.14 is the default in this image, but many packages don't have pre-compiled wheels for it yet (e.g., `pydantic-core` requires Rust compilation). **Python 3.12 has the best compatibility** with the current ML/AI ecosystem. This guide shows how to create a Python 3.12 virtual environment using yay.

### The Problem: `pacman -Syu` Resets Python

> **WARNING**: Even if you install Python 3.12 and set it as the default, **every `pacman -Syu` will upgrade Python to 3.14.x again** and potentially break the `python` / `python3` symlinks. Arch is a rolling release — this is expected behavior. The solution is to use **virtual environments** with explicit Python 3.12 binaries, so your projects are isolated from system Python changes.

### Option A: Python 3.12 venv via yay (recommended for Dockerfiles)

This is the cleanest approach for container images. It uses yay to install `python312` from AUR, then creates a virtual environment with explicit Python 3.12 paths.

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# 1) Install Python 3.12 from AUR via yay
#    MUST run as builder user (yay refuses root)
RUN su - builder -c 'yay -S --noconfirm python312'
# After this: /usr/bin/python3.12 is available

# 2) Create a virtual environment using Python 3.12 EXPLICITLY
#    The path /usr/bin/python3.12 never changes — immune to pacman updates
RUN /usr/bin/python3.12 -m venv /opt/app-venv

# 3) Install your dependencies inside the 3.12 venv
#    All packages use pre-compiled wheels for 3.12 (no Rust needed!)
RUN /opt/app-venv/bin/pip install --no-cache-dir \
    fastapi uvicorn pydantic pydantic-settings \
    httpx asyncpg redis qdrant-client

# 4) Use the venv python in your CMD/ENTRYPOINT
CMD ["/opt/app-venv/bin/python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

**Why this works:**

- `/usr/bin/python3.12` is a real binary, not a symlink — `pacman -Syu` never touches it
- The venv copies the Python 3.12 interpreter, so it's completely isolated
- All pip packages resolve to Python 3.12 pre-compiled wheels → no compilation needed
- Even if `pacman -Syu` upgrades system Python to 3.15 tomorrow, your venv stays on 3.12

### Option B: Python 3.12 venv with system packages (hybrid)

If you also want to use Arch's pre-compiled Python packages from pacman, create the venv with `--system-site-packages`:

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# 1) Install Python 3.12 from AUR
RUN su - builder -c 'yay -S --noconfirm python312'

# 2) Install Python packages via pacman (pre-compiled, no compilation)
RUN pacman -S --noconfirm --needed \
    python-fastapi python-pydantic python-httpx

# 3) Create venv with --system-site-packages
#    The venv sees both pacman packages AND pip packages
RUN /usr/bin/python3.12 -m venv --system-site-packages /opt/app-venv

# 4) pip only installs what pacman doesn't have
RUN /opt/app-venv/bin/pip install --no-cache-dir \
    uvicorn qdrant-client motor

CMD ["/opt/app-venv/bin/python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

**Note**: Some pacman Python packages may not be available for Python 3.12 (they target 3.14). In that case, the hybrid approach gracefully falls back to pip — see Option A for the safest path.

### Option C: Interactive (for development inside a running container)

```bash
# Enter container
podman run -it --rm ghcr.io/drmicalet/arch-base-privileged:latest bash

# Install Python 3.12 from AUR
su - builder -c 'yay -S --noconfirm python312'

# Verify
python3.12 --version    # Python 3.12.x

# Create venv
python3.12 -m venv /opt/myproject-venv
source /opt/myproject-venv/bin/activate

# Install dependencies (3.12 wheels — fast, no compilation)
pip install fastapi uvicorn pydantic torch

# Work normally
python main.py
```

### FAQ: Python Versions

| Question | Answer |
|---|---|
| What version is `python` / `python3`? | **3.14.x** (Arch system default) |
| Will `pacman -Syu` change this? | **Yes**, always — it's a rolling release |
| Can I have both 3.12 and 3.14? | **Yes** — install 3.12 via yay, both coexist |
| Which should I use for ML/AI? | **3.12** — better package compatibility |
| Which should I use for production? | **3.12** — more stable ecosystem |
| What if I need 3.14 features? | Use `python3.14` explicitly or a 3.14 venv |
| How do I protect against `pacman -Syu`? | Use **venvs with explicit `/usr/bin/python3.12`** path |

---

## Run with GPU (ROCm)

```bash
podman run -d \
  --name arch-rocm \
  --device /dev/kfd \
  --device /dev/dri \
  -e HSA_OVERRIDE_GFX_VERSION=10.3.0 \
  ghcr.io/drmicalet/arch-base-privileged:latest

# Verify GPU access
podman exec -it arch-rocm rocminfo
```

---

## What's Included

### System Packages

| Component | Purpose |
|---|---|
| `base`, `base-devel` | Core system + compilation toolchain (gcc, make, cmake, etc.) |
| `git` | Version control |
| `curl`, `wget` | Downloads |
| `sudo` | Privilege management (passwordless for root) |
| `systemd` | Service management |
| `vim`, `nano` | Text editors |
| `pacman` | Package manager (keyring pre-initialized) |

### AUR Helper

| Component | Purpose |
|---|---|
| `yay` | AUR package manager, pre-configured (`/usr/sbin/yay`) |
| `builder` user | Non-root user for yay (uid=1000, gid=1000, group wheel) |
| sudo access | `builder` can sudo without password |

### Python Stack

| Component | Version | Notes |
|---|---|---|
| `python` / `python3` | **3.14.x** | Arch system default |
| `python3.12` | 3.12.x | Available via AUR (`yay -S python312`) |
| `python3.14` | 3.14.x | Explicit access to system Python |
| `pip` | Latest | Package installer (for default Python) |

### Size Breakdown

| Component | Approx. Size | Purpose |
|---|---|---|
| Arch base + base-devel | ~915 MB | GCC, make, dev tools, systemd |
| Python 3.14 (system) | ~124 MB | Default system Python |
| Python 3.12 (AUR, on demand) | ~115 MB | ML/AI compatibility (install via yay) |
| yay + git | ~45 MB | AUR package installation |
| Other tools & libs | ~100 MB | DNS config, sudo, editors, utilities |
| **Total** | **~1.3 GB** | |

---

## Use Cases

### AI/ML Containers (with Python 3.12)

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# AMD GPU support
RUN pacman -S --noconfirm rocm-hip-sdk

# Python 3.12 for best ML compatibility
RUN su - builder -c 'yay -S --noconfirm python312'
RUN /usr/bin/python3.12 -m venv /opt/ml-venv
RUN /opt/ml-venv/bin/pip install torch torchvision
```

### Kubernetes / K3s Workloads

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: ghcr.io/drmicalet/arch-base-privileged:latest
    securityContext:
      privileged: true
```

### Development Environments

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# Install any AUR package easily
RUN su - builder -c 'yay -S --noconfirm neovim-nightly-bin'

# Python development
RUN su - builder -c 'yay -S --noconfirm python312'
RUN /usr/bin/python3.12 -m venv /opt/dev-venv
RUN /opt/dev-venv/bin/pip install fastapi uvicorn
```

### Database & Service Containers

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

RUN pacman -S --noconfirm postgresql redis valkey

# Run with systemd for proper service management
CMD ["/sbin/init"]
```

---

## Available Tags

| Tag | Description | Size |
|---|---|---|
| `latest` | Python 3.14 (default) + yay + builder + DNS fix | ~1.3 GB |
| `slim` | Same as latest | ~1.3 GB |

---

## Building from Source

```bash
git clone https://github.com/drmicalet/arch-base-privileged.git
cd arch-base-privileged
podman build --network=host -t arch-base-privileged:local .
```

> **Note**: Always use `--network=host` when building to ensure DNS resolution works during the build process.

---

## Contributing

Contributions are welcome! Please ensure:

1. All changes work with `--network=host` during build
2. yay commands always use `su - builder -c 'yay ...'` (never root)
3. The `builder` user (uid=1000) and sudo configuration are preserved
4. Document any Python version considerations clearly

---

## License

GPL-3.0-or-later — Same license as Arch Linux packages.

---

## Credits

Maintained by [drmicalet](https://github.com/drmicalet).

Built on Arch Linux.
