# ShellCrash (GHCR Image)

**ShellCrash GHCR 镜像**，用于在容器环境中运行 ShellCrash，支持 **HTTP / SOCKS 代理** 与 **旁路由透明代理** 两种部署模式。

`ghcr` 是基于 `dev` 创建的独立镜像发布分支，不向上游发起合并请求。该分支仅保留 GHCR 镜像构建工作流。

- 推送到 `ghcr` 分支会自动构建，也可在 GitHub Actions 中选择该分支手动运行 `Build and publish ShellCrash to GHCR`。
- 使用 GitHub 自动提供的 `GITHUB_TOKEN` 登录 GHCR，无需配置 Docker Hub 凭据。
- 镜像地址：`ghcr.io/cyxc1124/shellcrash`，发布 `latest`、`version` 文件中的版本号和 `sha-<完整提交 SHA>` 三种标签。
- 支持 `linux/386`、`linux/amd64`、`linux/arm64` 和 `linux/arm/v7`。
- 沿用当前分支的 `ShellCrash.tar.gz` 和 Dockerfile，内核、规则集和面板仍从上游下载。

------

## Quick Start（最小化运行）

仅启用 HTTP(S) / SOCKS5 代理功能，适用于基础代理需求，Mix代理端口：7890，面板管理端口：9999。

```shell
docker run -d \
  --name shellcrash \
  -p 7890:7890 \
  -p 9999:9999 \
  ghcr.io/cyxc1124/shellcrash:latest
```

------

## Container Management（容器管理）

首次部署完成后，请务必使用以下命令进入容器完成设置（导入配置文件，允许开机启动，及启动内核服务），之后也可用此命令进入容器sh环境进行管理：

```shell
docker exec -it shellcrash sh -l
```

------

## Advanced Usage（旁路由 / 透明代理）

适用于旁路由、软路由或需要透明代理的部署场景，需提前创建macvlan，这里不推荐使用容器的host模式。

### 1. 创建 macvlan 网络

此处请根据实际网络环境调整参数，如之前已创建可忽略。

```shell
docker network create \
  --driver macvlan \
  --subnet 192.168.31.0/24 \
  --gateway 192.168.31.1 \
  -o parent=eth0 \
  macvlan_lan
```

### 2. 启动容器（旁路由模式）

```shell
docker run -d \
  --name shellcrash \
  --network macvlan_lan \
  --ip 192.168.31.222 \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --cap-add SYS_ADMIN \
  --sysctl net.ipv4.ip_forward=1 \
  --device /dev/net/tun:/dev/net/tun \
  --restart unless-stopped \
  ghcr.io/cyxc1124/shellcrash:latest
```

### 3. 配置需要路由的设备

将需要路由的设备IPV4网关与DNS均指向启动容器时指定的IP地址如：192.168.31.222

注意，旁路由模式必须禁用子设备的IPV6地址，或主路由的IPV6功能，否则流量可能会经由IPV6直连而不会进入旁路由转发

------

## Persistent Configuration（持久化配置,可选）

推荐使用 volume 挂载以持久化 ShellCrash 配置。

### 1. 创建宿主机目录

```shell
mkdir -p /root/ShellCrash
```

### 2. 启用持久化

将命令粘贴到你的实际容器启动命令中间，例如：

```shell
docker run -d \
  ………………
  -v shellcrash_configs:/etc/ShellCrash/configs \
  ………………
```

------

## Compose Deployment（Compose部署）

### 1. 创建宿主机目录并进入目录

```shell
mkdir -p /tmp/ShellCrash
cd /tmp/ShellCrash
```

### 2. 下载Compose模版

```shell
curl -sSL https://raw.githubusercontent.com/cyxc1124/ShellCrash/ghcr/docker/compose.yml -O
```

### 3. 根据本地环境修改Compose模版

```shell
vi compose.yml #或者使用其他文本编辑器
```

### 4. 运行服务

```shell
docker compose up -d
```

------

## Delete（移除容器镜像或删除卷）

### Docker删除容器

```shell
docker rm -f shellcrash
```

### Docker删除卷

```shell
docker volume rm shellcrash_configs
```

### Compose删除容器&卷

```shell
docker-compose down -v
```

## Notes

- 内置公网防火墙功能无法管理宿主机网络，请自行做好宿主机7890/9999端口的网络防护！
- 旁路由模式需要宿主机支持 `TUN`
- macvlan 网络下宿主机默认无法直接访问容器 IP
- 透明代理场景可能需要额外的网络规划
