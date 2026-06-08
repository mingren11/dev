# dockerize.sh

一个轻量级的 Docker 开发容器管理脚本，支持自动构建、增量重建（Dockerfile 变更检测）和用户映射，专为本地开发工作流设计。

---

## 功能特性

- **自动命名**：以当前目录名作为容器名称（自动转小写）
- **变更检测**：通过 MD5 校验 Dockerfile，仅在变更时重建容器，避免无谓重建
- **用户映射**：将宿主机用户（UID/GID）映射进容器，避免文件权限问题
- **配置同步**：自动同步 `.gitconfig`、`.git-credentials`、bash 配置及 Go 工具链配置
- **快速进入**：容器已运行时直接 `exec` 进入，无需重启

---

## 目录结构要求

```
<your-project>/
├── artifacts/
│   └── docker/
│       └── dev.dockerfile      # 必须存在
├── dockerize.sh                 # 本脚本
└── docker_bashrc.sh             # 可选：注入容器的 bash 配置
```

---

## 快速开始

```bash
chmod +x dockerize.sh
./dockerize.sh
```

脚本会自动完成以下流程：

```
检查 Dockerfile 是否存在
      ↓
容器是否已存在？
  ├─ 是 → Dockerfile 有变更？
  │         ├─ 是 → 删除旧容器，重新构建并启动
  │         └─ 否 → 容器未运行则启动 → 直接进入
  └─ 否 → 构建镜像 → 启动容器 → 初始化配置 → 进入容器
```

---

## 容器运行参数

| 参数 | 说明 |
|------|------|
| `--privileged` | 特权模式（用于设备访问） |
| `--shm-size 2G` | 共享内存 2GB |
| `--gpus all` | 挂载所有 GPU |
| `-v $PWD:/<container_name>` | 项目目录挂载为容器工作目录 |
| `-v $HOME/.cache` | 宿主机缓存目录挂载（加速构建） |
| `/dev/bus/usb`, `/media`, `/mnt` | 设备与存储挂载 |
| X11 socket | 支持图形界面转发 |

---

## 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `BASH_LOCAL` | 自定义 bash 配置文件路径 | `<脚本目录>/docker_bashrc.sh` |
| `SSH_PORT` | 若设置，则在容器内启用 SSH 服务 | 未设置 |
| `ROOTLESS` | 启用 rootless 模式下的 Bazel 适配 | - |
| `BAZEL_WORKSPACE` | 标记为 Bazel 工作区，配合 `ROOTLESS` 使用 | - |

---

## 自动同步内容

脚本启动时会自动将以下宿主机配置同步进容器：

- `~/.gitconfig` 和 `~/.git-credentials` → git 操作免重复配置
- `docker_bashrc.sh` → 注入为容器内 `~/.bash_local`，进入容器时自动加载
- `~/.config/go` → Go 工具链配置（项目含 `go.mod` 时生效）

---

## 注意事项

- Dockerfile 路径固定为 `artifacts/docker/dev.dockerfile`，请确保该文件存在
- MD5 状态文件存储于 `/tmp/.<container_name>_dockerfile_md5`，重启系统后会触发一次重建
- 以 root 身份运行宿主机时，容器内同样以 root 运行，跳过用户映射
- `--gpus all` 需要宿主机安装 NVIDIA Container Toolkit，无 GPU 环境请自行移除该参数

## 可能出现的问题

实际过程当中可能会出现CDI 配置问题，安装/重装 NVIDIA Container Toolkit
```
# Ubuntu/Debian
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit

# 配置 Docker runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

```

---

## 作者

renming ·shuaiqiduoyi@gmail.com
