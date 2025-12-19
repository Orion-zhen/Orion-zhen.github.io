---
date: "2025-09-30T09:02:03+08:00"
draft: false
title: "容器管理和 NAS 方案"
slug: "dockge-and-nas"
description: "使用 Dockge"
image: /post/tutorial/dockge-and-nas/cover.jpg
categories:
    - 教程
tags:
    - docker
    - 容器
license: GNU General Public License v3.0
---

> [封面出处](https://www.pixiv.net/artworks/68148178)

## 为什么

### 选择 Dockge

Dockge 最大的优势就在于, 它不会将容器配置或者 `compose` 文件绑架到私有数据库/目录中, 这些文件以原生的形式存放在宿主机文件系统的自定义目录中. Dockge 只是一个对 `compose` 命令美丽的 UI 封装.

## 配置方案

### docker

使用 docker 而非更新的 podman 的原因在于, 这是作为服务而运行的, 需要在重启后能自动上线. podman 因为没有采用 C/S 架构, 导致必须手动将其加入到 systemd 中, 并且在重启前后的操作也比较麻烦. 与此相反, docker 因为具备守护进程, 所以在自动上线提供服务这方面强于 podman.

首先安装 docker:

```bash
sudo pacman -U docker docker-compose docker-buildx
```

然后将当前用户加入 docker 组中:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

最后启动 docker 服务:

```bash
sudo systemctl enable --now docker.service docker.socket
```

### Dockge

首先创建两个目录 `/opt/dockge` 和 `/opt/stacks`, 并将其所有权授予当前用户:

```bash
sudo mkdir -p /opt/dockge /opt/stacks
sudo chown -R $USER /opt/dockge
sudo chown -R $USER /opt/stacks
```

然后获取 Dockge 的 `compose.yaml` 文件并放在 `/opt/dockge` 目录下:

```bash
curl https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml --output compose.yaml
```

可以自行编辑一些选项. 例如可以手动指定来自镜像的 image:

```yaml
services:
    image: docker.1ms.run/louislam/dockge:1
```

以及启用终端:

```yaml
environment:
    - DOCKGE_ENABLE_CONSOLE=true
```

接着启动 Dockge:

```bash
docker-compose up -d
```

启动成功后, 可以在 5001 端口访问 Dockge 网页端. 如果嫌登录账号麻烦的话, 可以在 设置 -> 安全 中禁用验证.

### 添加服务

首先在 Dockge 页面中添加容器栈, 可以指定从镜像源获取容器映像. 接着选择保存但不启动, 因为此时一般还没有配置挂载目录.

所有的容器数据目录都应该放在对应的容器栈文件夹中. 先创建对应的数据目录, 如 `/opt/stacks/<stack-name>/data`, 然后再在 compose 文件中编辑对应的挂载项.

当一切配置妥当, 可以启动容器栈. 不久就可以正常访问了.

值得注意的是, docker 容器内要访问宿主机的网络, `host.docker.internal` 在 Linux 下是不行的, 必须显式地指定 `172.17.0.1`

### 硬盘自动挂载

使用 `systemd.automount` 作为自动挂载的方案.

首先找到硬盘的 UUID:

```bash
lsblk -f
```

然后创建想要挂载的目录:

```bash
sudo mkdir -p /mnt/db0
```

接着创建 `mount` 单元文件. 注意文件名必须和挂载路径完全对应. 例如挂载点 `/mnt/db0` 对应的文件名就是 `mnt-db0.mount`. 创建文件 `/etc/systemd/system/mnt-db0.mount`:

```toml
[Unit]
Description=Mount for db0

[Mount]
What=/dev/disk/by-uuid/<UUID>
Where=/mnt/db0
Type=ntfs # 根据硬盘具体的文件类型
Options=defaults,noatime # 挂载选项, noatime 可以提高性能

[Install]
WantedBy=multi-user.target
```

再创建 `automount` 单元文件. 文件名必须和 `mount` 文件匹配. 创建文件 `/etc/systemd/system/mnt-db0.automount`:

```toml
[Unit]
Description=Automount for db0

[Automount]
Where=/mnt/db0
TimeoutIdleSec=15m # 可选, 如果 15 分钟没有访问, 自动卸载

[Install]
WantedBy=multi-user.target
```

最终, 启动 `automount` 服务. 注意, 只需要启动 `automount`, 它会自动管理 `mount` 服务:

```bash
sudo systemctl enable --now mnt-db0.automount
```

**⚠️注意**: 尤其要保证文件名, 挂载路径要一一对应!

### 网络

我使用 OpenWrt 作为软路由, 本地服务器的主机名为 `aio.lan`. 要配置所有 `*.aio.lan` 的域名指向服务器, 只需要在 OpenWrt 的设置界面中找到 网络 -> DNS -> 常规 页面中的 **地址** 项.

其语法是: `/doman.example/ip-addr`, 会将所有匹配 `domain.example` 的域名解析到 `ip-addr` 上.

例如我的配置是: `/aio.lan/192.168.114.114`, 会将所有对 `*.aio.lan` 的域名解析到本地服务器, 以便 `nginx` 做反向代理.

## 应用服务

### Immich

#### 存储模板

Immich 支持用特定格式的目录来存储文件. 这里我使用的格式是:

```text
{{#if album}}{{album}}{{else}}散件/{{/if}}/{{y}}/{{MM}}/{{filename}}
```

#### 手机备份

在手机 App 备份时, 可以选择打开 设置 -> 备份选项 -> 同步相册. 这样就可以在 immich 中同步手机系统相册的划分了.

#### 机器学习

Immich 支持了新的 nllb CLIP 模型, 可以支持中文搜索图片. 要使用自定义模型, 需要修改 `immich-machine-learning` 容器的挂载目录.

获取最新的 `hwaccel.ml.yml` 文件并放置于 `compose.yaml` 同级目录下, 即可启用硬件加速的 AI 功能:

```bash
wget -O hwaccel.ml.yml https://github.com/immich-app/immich/releases/latest/download/hwaccel.ml.yml
```

```yaml
immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:release-cuda
    extends: # uncomment this section for hardware acceleration - see https://immich.app/docs/features/ml-hardware-acceleration
      file: hwaccel.ml.yml
      service: cuda # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - <model-cache>:/cache
```

CLIP 模型应满足的目录结构为 `/cache/clip/<model name>`. 可以提前在宿主机环境中下载好模型到本地, 然后挂载进 immich 容器中. 这里推荐专门为多语言设计的模型 [nllb-clip-large-siglip__v1](https://huggingface.co/immich-app/nllb-clip-large-siglip__v1), 完成下载后只需在 immich 管理设置 -> 机器学习设置 -> 智能搜索 中填入对应的模型名即可.

#### 迁移数据库

Immich [将文件路径保存在数据库中](https://github.com/immich-app/immich/discussions/3299), 它不会扫描库文件夹来更新数据库, 因此备份非常重要.

**备份**:

首先进入到存在 `compose.yaml` 文件存在的文件夹, 然后执行命令:

```bash
docker compose exec -T database pg_dumpall --clean --if-exists -U postgres | gzip > immich_db_backup.sql.gz
```

这行命令的意思是, 让 `docker` 在 `compose.yaml` 文件中找到 `database` 对应的容器, 进入其中并执行 `pg_dumpall ...` 命令. `-U` 参数指定了数据库用户, 一般来说是默认的 `postgres`. 最终在导出数据后将其压缩为一个压缩包.

**恢复**:

将上一步中得到的压缩包移动到新设备上的 `compose.yaml` 文件所在的目录. 首先运行 `docker compose up -d` 启动一个全新的 immich 实例, 然后在当前目录下执行:

```bash
gunzip --stdout immich_db_backup.sql.gz | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" | docker compose exec -i database psql -U postgres
```

其中 `sed` 命令的作用在于避免因为某些插件在全新的实例中不存在而导致的恢复失败的情况.

### 音乐服务

使用 Navidrome + music-tag-web + symforium 来实现云音乐播放.

选用 Navidrome 的原因在于, 它是一个专注于音乐管理的方案, 且提供了 Subsonic API 兼容的服务, 可以对接许多 Subsonic App. Navidrome 本身完全依靠音乐文件中内置的元信息来整理音乐, 完全不管实际的目录结构如何. 所以一个能管理音乐元数据的方案就尤为必要. 这里选择 music-tag-web 来作为音乐文件的打标工具. 最后需要一个手机 App 来连接 Navidrome, 这里我选择需要付费 ~6$ 的 symforium, 它提供了很多强大的功能, 且 UI 美观.

#### Navidrome

```yaml
services:
  core:
    environment:
      - TZ=Asia/Shanghai
      - ND_LOGLEVEL=info
      - ND_DEFAULTLANGUAGE=zh-Hans # 默认语言改为中文
      - ND_ENABLEGRAVATAR=false # 禁用 Gravatar 头像集成
      - ND_ENABLEREPLAYGAIN=false # 禁用 Web 界面调整回放增益的功能
      - ND_ENABLESHARING=false # 禁用分享功能，如果你需要分享给别人可以打开
      - ND_ENABLESTARRATING=false # 禁用 Web 界面的五星评级歌曲功能
      - ND_ENABLEUSEREDITING=false # 禁止普通用户更改自身信息与登录凭据，安全考量
      - ND_LASTFM_ENABLED=false # 禁用 Last.fm集成
      - ND_LISTENBRAINZ_ENABLED=false # 禁用 ListenBrainZ 元数据库集成
      - ND_MAXSIDEBARPLAYLISTS=300 # 调整侧边最多显示的播放列表数为 300（默认 100）
      - ND_SCANNER_GROUPALBUMRELEASES=true # 禁用按照日期区分专辑，防止 Navidrome 错误的按照日期从专辑中拆分单曲
    image: docker.1ms.run/deluan/navidrome:latest
    ports:
      - 5006:4533
    restart: unless-stopped
    volumes:
      - ./data:/data
      - 音乐目录:/music:ro # 注意音乐文件夹是ro即只读的，Navidrome 不需要也不会对你的音乐库实际文件进行任何修改操作
```

#### music-tag-web

> 现在最新 V1 版本的 music-tag-web 调用网易 API 时存在 bug, 所以回退到 2.5.6

```yaml
services:
  music-tag:
    image: docker.1ms.run/xhongc/music_tag_web:2.5.6
    container_name: music-tag-web
    ports:
      - 5005:8002
    volumes:
      - 音乐目录:/app/media:rw
      - ./data:/app/data
    restart: unless-stopped
networks: {}
```

#### symforium

添加 (Open) Subsonic 类型的媒体源即可. 值得注意的是, 存在内网和外网可以要使用不同的服务器 URL 的情况. 当你在家里, 跟 Navidrome 服务器处在同一个网络环境下时, 可以直接通过 IP 访问服务器, 而当你在外面时才通过公网域名去访问. 这一点可以通过配置媒体源的**次要连接**来实现. 在**主要连接**中填写内网 IP, 在**次要连接**中填写公网域名. 这样就可以做到在内网下优先使用 IP, 仅当 IP 不可用时才会切换到次要连接的公网域名. 不过切换需要一定时间, 这个时间内无法与服务器进行同步.
