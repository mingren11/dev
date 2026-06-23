# dev

A lightweight Docker development container manager with automatic builds, optional rebuilds, and host user mapping for local development workflows.

---

## Features

- **Auto naming**: Uses the current directory name as the container name (lowercased)
- **Manual rebuild**: Reuses an existing container by default; rebuild only with `--rebuild` or `-r`
- **User mapping**: Maps the host user (UID/GID) into the container to avoid file permission issues
- **Shared Git auth**: Mounts `~/.ssh`, `~/.gitconfig`, and `~/.git-credentials` so pull/push works on the host and in the container
- **Config bootstrap**: Copies bash and Go toolchain config into the container on first setup
- **Fast entry**: `exec`s into a running container without restarting it

---

## Required layout

```
<your-project>/
├── artifacts/
│   └── docker/
│       └── dev.dockerfile      # required
└── dev                           # this script
```

---

## Quick start

```bash
chmod +x dev
dev
```

Add to system PATH:
```
echo "export PATH="$PATH:<your-dev-path>"" > ~/.bashrc
echo "export PATH="$PATH:<your-dev-path>"" > ~/.zshrc

```

To remove an existing container and rebuild from scratch:

```bash
dev --rebuild
# or
dev -r
```

The script follows this flow:

```
Check that dev.dockerfile exists
      ↓
Does the container already exist?
  ├─ yes → Was --rebuild / -r passed?
  │         ├─ yes → remove old container, rebuild, and start
  │         └─ no  → start if stopped → enter container
  └─ no  → build image → start container → bootstrap config → enter container
```

---

## Container runtime options

| Option | Description |
|--------|-------------|
| `--privileged` | Privileged mode for device access |
| `--network host` | Host network (required for some GUI and service setups) |
| `--shm-size 2G` | 2 GB shared memory |
| `--gpus all` | Expose all GPUs |
| `-v $PWD:/<container_name>` | Mount project directory as the working directory |
| `-v $HOME/.cache` | Mount host cache directory (faster builds) |
| `-v $HOME/.ssh` | Mount SSH keys and `known_hosts` (if present) |
| `-v $HOME/.gitconfig` | Mount Git config (if present) |
| `-v $HOME/.git-credentials` | Mount Git HTTPS credentials (if present) |
| `/dev/bus/usb`, `/media`, `/mnt` | Device and storage mounts |
| X11 socket | GUI forwarding via `$DISPLAY` |

Git auth paths are bind-mounted at container creation time. Changes on the host are visible inside the container immediately, and vice versa.

---

## Environment variables

| Variable | Description | Default |
|----------|-------------|---------|
| `BASH_LOCAL` | Path to a custom bash config file | `<script-dir>/docker_bashrc.sh` |
| `SSH_PORT` | If set, enables an SSH server inside the container | unset |
| `ROOTLESS` | Enable Bazel settings for rootless mode | - |
| `BAZEL_WORKSPACE` | Mark as a Bazel workspace; used with `ROOTLESS` | - |

---

## Bootstrap vs mounted config

On first container setup, the script copies these host files into the container:

- `docker_bashrc.sh` → `~/.bash_local` (sourced from `~/.bashrc`)
- `~/.config/go` → Go toolchain config (only when the project contains `go.mod`)

These paths are bind-mounted when the container is created (existing containers need a rebuild to pick up new mounts):

- `~/.ssh` → SSH keys and `known_hosts`
- `~/.gitconfig` → Git user and remote settings
- `~/.git-credentials` → HTTPS credential store

---

## Notes

- The Dockerfile path is fixed at `artifacts/docker/dev.dockerfile`
- Editing `dev.dockerfile` does not rebuild automatically; run `dev --rebuild` or `dev -r`
- Changing volume mounts (for example Git/SSH paths) also requires `dev --rebuild` or `dev -r`
- When the host is run as root, the container also runs as root and skips user mapping
- `--gpus all` requires the NVIDIA Container Toolkit on the host; remove it on machines without a GPU
- Keep `~/.ssh` at mode `700` and private keys at `600`, or SSH may refuse to use them

---

## Troubleshooting

### NVIDIA / CDI issues

If you hit CDI configuration errors, install or reinstall the NVIDIA Container Toolkit:

```bash
# Ubuntu/Debian
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit

# Configure Docker runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### GUI / display issues

On the host, allow local X11 connections:

```bash
xhost +local:
```

If the display still fails, confirm the container uses host networking (`--network host` is enabled by default).

### Git auth inside the container

After rebuilding, verify SSH access:

```bash
ssh -T git@github.com
git pull
```

---

## Add `dev` to your PATH

Add the script directory to `~/.bashrc` or `~/.zshrc`:

```bash
export PATH="$PATH:/path/to/dev"
```

---

## Author

renming · shuaiqiduoyi@gmail.com
