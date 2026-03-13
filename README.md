# Arch Base Privileged

Imagen base Arch Linux con privilegios completos, diseñada para contenedores que necesitan acceso total al sistema, systemd, o ejecutar servicios complejos.

## Características

- **Arch Linux rolling** - Siempre actualizado con los últimos paquetes
- **Sudo sin contraseña** - Usuario con privilegios completos de administrador
- **Systemd habilitado** - Posibilidad de ejecutar servicios dentro del contenedor
- **Herramientas de desarrollo completas** - GCC, make, cmake, git, curl, wget
- **ROCm ready** - Preparada para contenedores con GPU AMD
- **Base optimizada** - Solo ~1.5 GB

## Uso en Dockerfile

```dockerfile
FROM ghcr.io/drmicalet/arch-base-privileged:latest

# Actualizar e instalar lo que necesites
RUN pacman -Syu --noconfirm && \
    pacman -S --noconfirm \
    python \
    nodejs \
    npm \
    tu-paquete

# Comando por defecto
CMD ["/bin/bash"]
```

## Descarga

```bash
# Desde GitHub Container Registry
docker pull ghcr.io/drmicalet/arch-base-privileged:latest

# Con Podman
podman pull ghcr.io/drmicalet/arch-base-privileged:latest

# Importar en K3s/Kubernetes
podman save -o arch-base-privileged.tar ghcr.io/drmicalet/arch-base-privileged:latest
sudo k3s ctr images import arch-base-privileged.tar
```

## Ejecutar con systemd

Para contenedores que necesitan ejecutar servicios:

```bash
docker run -d \
  --name arch-systemd \
  --privileged \
  --tmpfs /tmp \
  --tmpfs /run \
  --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  ghcr.io/drmicalet/arch-base-privileged:latest \
  /sbin/init

# Luego ejecutar comandos en el contenedor
docker exec -it arch-systemd systemctl status
docker exec -it arch-systemd pacman -S nginx
docker exec -it arch-systemd systemctl enable --now nginx
```

## Ejecutar con GPU AMD (ROCm)

```bash
docker run -d \
  --name arch-rocm \
  --device /dev/kfd \
  --device /dev/dri \
  -e HSA_OVERRIDE_GFX_VERSION=10.3.0 \
  ghcr.io/drmicalet/arch-base-privileged:latest

# Verificar GPU
docker exec -it arch-rocm rocminfo
```

## Contenido Incluido

| Componente | Propósito |
|------------|-----------|
| base-devel | Compilación de software (gcc, make, etc.) |
| git | Control de versiones |
| curl, wget | Descargas |
| sudo | Elevación de privilegios |
| systemd | Gestión de servicios |
| vim, nano | Editores de texto |

## Diferencias con otras imágenes Arch

| Característica | archlinux oficial | arch-base | arch-base-privileged |
|----------------|-------------------|-----------|---------------------|
| Tamaño | ~544 MB | ~1.5 GB | ~1.5 GB |
| sudo sin password | ❌ | ❌ | ✅ |
| systemd | ❌ | ❌ | ✅ |
| base-devel | ❌ | ✅ | ✅ |
| Para servicios | ❌ | ❌ | ✅ |
| Para desarrollo | Básico | Medio | Completo |

## Casos de Uso Ideales

- **Contenedores de desarrollo** con herramientas completas
- **Servicios que necesitan systemd** (bases de datos, servidores web)
- **Contenedores con GPU** (ROCm, CUDA)
- **CI/CD pipelines** que requieren acceso completo
- **Testing de software** en Arch Linux

## Mantener Actualizado

Esta imagen se actualiza periódicamente. Para tener la última versión:

```bash
# Actualizar la imagen base
docker pull ghcr.io/drmicalet/arch-base-privileged:latest

# Si la usas como base, reconstruir tus imágenes
docker build --no-cache -t mi-imagen .
```

O actualizar dentro del contenedor:

```dockerfile
# En tu Dockerfile, siempre actualizar
RUN pacman -Syu --noconfirm
```

## Licencia

GPL-3.0

Arch Linux y sus paquetes mantienen sus propias licencias.

## Autor

**Drmicalet** - [GitHub](https://github.com/Drmicalet)

---

*Base optimizada para contenedores con necesidades avanzadas*
