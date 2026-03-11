# arch-base-privileged

![Architecture](https://img.shields.io/badge/Architecture-x86__64-blue)
![Base](https://img.shields.io/badge/Base-Arch_Linux-1793D1)
![License](https://img.shields.io/badge/License-GPL--3.0--or--later-blue)

A **minimal, privileged Arch Linux base image** designed for advanced container orchestration, AI/ML workloads, and systems requiring elevated permissions. This image extends the official Arch Linux base with critical improvements for production environments.

---

## Why This Image?

### 🚀 Advantages Over Official Arch Images

| Feature              | Official Arch         | arch-base-privileged                                   |
| -------------------- | --------------------- | ------------------------------------------------------ |
| **DNS Resolution**   | May fail in builds    | ✅ Improved DNS with localhost fallback                 |
| **Privileged Mode**  | Not configured        | ✅ Pre-configured for privileged operations             |
| **AUR Helper (yay)** | Not included          | ✅ Pre-installed with **builder user** (not root!)      |
| **Python Version**   | Python 3.14 only      | ✅ Python 3.12 + 3.14 (both available, 3.12 as default) |
| **Pacman Keyring**   | Manual setup required | ✅ Pre-initialized and trusted                          |
| **Build Network**    | Default only          | ✅ Localhost registry included                          |
| **Size**             | ~450MB                | ~1.3GB (optimized with essential tooling)              |

---

## Key Features

### 1. 🔧 Improved DNS Resolution

```dockerfile
# DNS works reliably during builds
RUN pacman -Syu --noconfirm  # No more "failed to synchronize databases"
```

Configured with localhost fallback and proper resolver settings for container environments.

### 2. 👤 Yay with Builder User (Security Best Practice)

```dockerfile
# yay runs as non-root user (required by yay)
RUN su - builder -c 'yay -S python312 --noconfirm'
```

**yay cannot run as root.** This image includes a dedicated `builder` user with sudo privileges, following Arch best practices for AUR package installation.

### 3. 🐍 Python 3.12 + Python 3.14 (Both Available)

Arch Linux ships with **Python 3.14** as the system default. However, many ML/AI packages and production environments require **Python 3.12** for compatibility.

This image provides **both versions**, with Python 3.12 set as the default:

```bash
# Default python is now 3.12
$ python --version
Python 3.12.x

# But Python 3.14 is still available
$ python3.14 --version  
Python 3.14.x

# Both coexist without conflicts
$ ls /usr/bin/python*
/usr/bin/python -> /usr/bin/python3.12
/usr/bin/python3 -> /usr/bin/python3.12
/usr/bin/python3.12
/usr/bin/python3.14
```

**Why Python 3.12 as default?**

- Better compatibility with PyTorch, TensorFlow, and ML libraries
- Stable ecosystem with well-tested packages
- Many packages don't yet support Python 3.14
- You can still use `python3.14` explicitly when needed

**How it works:**

- Python 3.14 remains installed (Arch's system Python)
- Python 3.12 installed from AUR alongside
- Symlinks `/usr/bin/python` and `/usr/bin/python3` point to Python 3.12
- No deletion or replacement of system Python

### 4. 🔑 Pre-initialized Pacman Keyring

```dockerfile
# No need to initialize keyring manually
RUN pacman -S package-name --noconfirm  # Works immediately
```

The Arch keyring is pre-initialized and trusted, saving build time and avoiding signature errors.

### 5. 🏠 Localhost Registry Support

Built-in support for localhost container registries, essential for:

- Local development workflows
- K3s / K3d clusters
- Air-gapped environments

### 6. ⚡ Privileged Container Ready

Pre-configured `securityContext` compatibility for:

- Hardware access (GPU, USB devices)
- System-level operations
- K3s/Kubernetes privileged pods

---

## Quick Start

### Pull the Image

```bash
docker pull ghcr.io/drmicalet/arch-base-privileged:latest
# or
podman pull ghcr.io/drmicalet/arch-base-privileged:latest
```

### Basic Usage

```bash
# Interactive shell
podman run -it --rm ghcr.io/drmicalet/arch-base-privileged:latest

# With privileged mode (for hardware access)
podman run -it --rm --privileged ghcr.io/drmicalet/arch-base-privileged:latest
```

### Example Containerfile

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# System packages work immediately (DNS + keyring configured)
RUN pacman -S --noconfirm git curl base-devel

# AUR packages with builder user
RUN su - builder -c 'yay -S --noconfirm some-aur-package'

# Python 3.12 is the default
RUN python -m pip install your-packages

# If you need Python 3.14 specifically
RUN python3.14 -m pip install other-packages

WORKDIR /app
CMD ["/bin/bash"]
```

---

## Available Tags

| Tag      | Description                    | Size   |
| -------- | ------------------------------ | ------ |
| `latest` | Stable with Python 3.12 + 3.14 | ~1.3GB |
| `slim`   | Same as latest                 | ~1.3GB |

---

## What's Included

### System Packages

- `base`, `base-devel` - Core system
- `git`, `curl`, `wget` - Essential tools
- `sudo` - Privilege management
- `pacman` - Package manager with initialized keyring

### AUR Helper

- `yay` - Pre-installed and configured
- `builder` user - For secure AUR package installation

### Python Stack

- `python312` - Python 3.12.x from AUR (default)
- `python` (3.14) - Arch's system Python (still available)
- `pip` - Package installer
- Symlinks: `/usr/bin/python` → `/usr/bin/python3.12`

---

## Size Breakdown

| Component | Size | Purpose |
|-----------|------|---------|
| archlinux:base-devel base | ~915 MB | GCC, make, dev tools |
| Python 3.12 | ~115 MB | ML/AI compatibility |
| Python 3.14 | ~124 MB | Latest Python version |
| yay + git | ~45 MB | AUR package installation |
| Other tools | ~100 MB | systemd, libraries, utilities |
| **Total** | **~1.3 GB** | |

**Why larger than archlinux:latest?** This image includes everything needed to build and install packages from AUR out of the box. No additional setup required.

---

## Use Cases

### 🤖 AI/ML Containers

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

RUN pacman -S --noconfirm rocm-hip-sdk  # AMD GPU support
RUN python -m pip install torch torchvision  # Uses Python 3.12
```

### 🐳 Kubernetes/K3s Workloads

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: ghcr.io/drmicalet/arch-base-privileged:latest
    securityContext:
      privileged: true  # Fully supported
```

### 🔧 Development Environments

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# Install any AUR package easily
RUN su - builder -c 'yay -S --noconfirm neovim-nightly-bin'
```

---

## Comparison with Alternatives

| Image                                    | Privileged | yay  | Python 3.12 | Python 3.14 | DNS Fixed | Size   |
| ---------------------------------------- | ---------- | ---- | ----------- | ----------- | --------- | ------ |
| `archlinux:latest`                       | ❌          | ❌    | ❌           | ✅           | ❌         | ~450MB |
| `archlinux:base-devel`                   | ❌          | ❌    | ❌           | ✅           | ❌         | ~915MB |
| `manjarolinux/base`                      | ❌          | ❌    | Varies      | Varies      | ❌         | ~700MB |
| `ghcr.io/drmicalet/arch-base-privileged` | ✅          | ✅    | ✅ (default) | ✅           | ✅         | ~1.3GB |

*Size includes all tooling for immediate productivity.*

---

## Building from Source

```bash
git clone https://github.com/drmicalet/arch-base-privileged.git
cd arch-base-privileged
podman build --network=host -t arch-base-privileged:local .
```

---

## Contributing

Contributions welcome! Please ensure:

1. All changes work with `--network=host`
2. yay commands use `su - builder -c`
3. Python 3.12 remains as default while keeping 3.14 available

---

## License

GPL-3.0-or-later - Same license as Arch Linux packages. See [LICENSE](LICENSE) for details.

---

## Credits

Maintained by [drmicalet](https://github.com/drmicalet).

Built on [Arch Linux](https://archlinux.org/) with ❤️
