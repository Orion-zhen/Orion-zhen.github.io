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

## 存储

### 文件格式

我选用 BtrFS. 因为它是较现代的, Linux 原生支持的文件系统, 也被群晖等系统作为默认的文件系统. 相比于 ZFS, 它不需要经过额外的 dkms 编译, 同时具备大部分 ZFS 的功能(如快照, 校验和, 压缩等), 且对内存的压力相对较小, 适合个人用户使用.

### 格式化硬盘

在开始前检查需要格式化的硬盘的挂载情况和设备号, 再三确定你要格式化的就是那块硬盘, 别搞错了:

```shell
lsblk -f
```

格式化 btrfs 需要用到 `btrfs-prog` 软件包:

```shell
sudo apt install btrfs-progs
```

以 `/dev/sda` 硬盘为例. 首先卸载所有的分区:

```shell
sudo umount /dev/sda1 # 如果有多个分区, 都要全部卸载
```

然后删除旧的数据和分区:

```shell
sudo wipefs -a /dev/sda
```

接下来创建新的 GPT 分区表. 尽管 btrfs 可以直接格式化整个裸盘, 但为了兼容性和管理方便, 还是先创建分区表然后格式化分区:

```shell
sudo parted /dev/sda mklabel gpt
```

创建一个占据 100% 空间的分区:

```shell
sudo parted /dev/sda mkpart primary btrfs 0% 100%
```

再次使用 `lsblk` 查看硬盘, 应该能看到 `/dev/sda` 下面多了一个分区 `/dev/sda1`.

分区创建成功, 接下来将这个分区格式化为 btrfs:

```shell
sudo mkfs.btrfs -f -L "label" /dev/sda1
```

这时分区应该格式化完成, 可以挂载了. 创建一个临时挂载点来检查结果:

```shell
sudo mkdir -p /mnt/tmp
sudo mount /dev/sda1 /mnt/tmp
```

检查挂载:

```shell
df -h | grep label
```

如果看到输出, 则表明挂载成功.

### 硬盘自动挂载

使用 `systemd.automount` 作为自动挂载的方案.

首先找到硬盘的 UUID:

```bash
lsblk -f
```

然后创建想要挂载的目录:

```bash
sudo mkdir -p /mnt/nas0
```

接着创建 `mount` 单元文件. 注意文件名必须和挂载路径完全对应. 例如挂载点 `/mnt/nas0` 对应的文件名就是 `mnt-nas0.mount`. 创建文件 `/etc/systemd/system/mnt-nas0.mount`:

```toml
# file: /etc/systemd/system/mnt-nas0.mount
[Unit]
Description=Mount nas0
# 只有在本地文件系统准备好后才尝试挂载
After=local-fs.target

[Mount]
What=/dev/disk/by-uuid/<UUID>
Where=/mnt/nas0
Type=btrfs
# comperss=zstd: 智能透明压缩
# noatime: 读取文件时不写入访问时间, 延长机械硬盘寿命
# space_cache=v2: 提升大容量硬盘空闲空间索引性能
Options=defaults,compress=zstd,noatime,space_cache=v2

[Install]
WantedBy=multi-user.target
```

**⚠️注意**: 尤其要保证文件名, 挂载路径要一一对应!

### 池化存储

我有两块硬盘, 我想将它们合并为一个逻辑上的大硬盘, 但又不希望它们就此被绑死在一起, 我希望这两块硬盘拆开后都可以分别独立地使用. 根据这种需求, 我不能使用 RAID, 因为 RAID 把两块盘绑在一起, 任何一块单独拿出来都是不能用的, 而且任何一块盘坏了, 整个 RAID 阵列就会出问题. 我也不能使用 btrfs 原生的扩容方案, 因为同样会将两块盘绑在一起.

最终我选择的是存储池化方案, 由 MergerFS 驱动. MergerFS 并不是一个文件系统, 而是一个抽象层, 将底层的若干个硬盘中的文件汇总展示在一处, 就好像是一个整体的硬盘一样.

仿照前面的硬盘的 mount 单元, 为 MergerFS 也创立一个单元文件 `/etc/systemd/system/mnt-pool.mount`:

```toml
# file: /etc/systemd/system/mnt-pool.mount
[Unit]
Description=MergerFS Pool Storage
# 需要所有子硬盘都已经挂载成功
Requires=mnt-nas0.mount mnt-nas1.mount
After=mnt-nas0.mount mnt-nas1.mount

[Mount]
# 语法为 /path1:/path2:/path3
What=/mnt/nas0:/mnt/nas1
# 合并后的挂载点
Where=/mnt/pool
Type=fuse.mergerfs

# 挂载参数
# defaults,allow_other: 允许非 root 用户访问 FUSE 文件系统
# use_ino: 让 MergerFS 提供固定的 inode 号, 避免被误认为是新文件
# cache.files=partial: 启用部分缓存, 提升读取性能
# dropcacheonclose=true: 关闭文件时释放缓存, 释放内存
# category.create=epmfs: 新文件写入策略, 优先寻找已经存在的路径, 否则找空闲最多的硬盘
# minfreespace=750G: 对 8TB 硬盘, 设置最后 750G 为保留空间, 以便维护 btrfs 文件系统
# fsname=mergerfs: 让 df -h 显示的名字更好看
Options=defaults,allow_other,use_ino,cache.files=partial,dropcacheonclose=true,category.create=epmfs,minfreespace=750G,fsname=mergerfs

[Install]
WantedBy=multi-user.target
```

有两点配置需要说明:

- `category.create`: 新文件存放策略. 我希望属于同一个文件夹的文件不要那么稀碎, 换言之, 我不希望例如一个相册里的相片分散在不同的硬盘里. `epmfs` 的策略是 Existing Path, Most Free Space, 即当你写入一个文件时, MergerFS 会检查这个文件所在的文件夹是否已经存在于某块物理硬盘上, 如果在, 则直接写入, 完全无视这块硬盘还剩下多少空间. 如果不是, 则回退到 `mfs` 策略, 寻找剩余空间最大的硬盘来创建文件.
- 使用 `epmfs` 策略后, 如果要精细调控文件去向, 可以手动在底层的硬盘挂载目录里创建对应的文件夹, 让 MergerFS 发现在某个硬盘上已经存在对应目录, 并写入文件.
- `minfreespace`: 当剩余可用空间不足这个数值时, MergerFS 将不再往这里写入新数据(除非 `epmfs` 策略指定). 这里我将其设置为 750GB 以适配我的 8TB 硬盘. 一般地, 对 btrfs 这种系统, 需要留出大约 10%~20% 的剩余空间, 用于 CoW 和 Balance 等操作.

最后使用 `systemd.automount` 单元文件来自动挂载 MergerFS 存储池. 创建 `/etc/systemd/system/mnt-pool.automount`:

```toml
# file: /etc/systemd/system/mnt-pool.automount
[Unit]
Description=Automount MergerFS Pool

[Automount]
Where=/mnt/pool
# 用不自动卸载, 减少磁头移动, 提升硬盘寿命
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

最后启动服务:

```shell
sudo systemctl enable --now mnt-pool.automount
```

### 硬盘维护

**1. 去重写**:

当一个目录的使用方式满足如下特征时, 应关闭 CoW:

- 数据库特征: 不重写整个文件, 而是频繁地打开一个或几个文件, 跳转到中间某一部分进行修改. 如 PostgreSQL, SQLite, Redis 等的数据库文件和工作文件.
- 虚拟机磁盘特征: 虚拟机内部的文件系统在运作, 而在宿主机看来只是在频繁修改一个巨大的 `.qcow2` 或 `vhdx` 文件. 如 KVM, Qemu, VMWare 等虚拟机软件的磁盘镜像文件.
- 预分配下载特征: 在软件开始下载前, 直接在磁盘上预先分配好一段区域, 然后通过多线程在这个区域内并发地下载. 如 BT/PT 下载器的临时目录.
- 缩略图缓存: Immich, Jellyfin 等服务会生成很多张小缩略图. 这个过程最好放在 SSD 上, 如果必须放在 HDD 上, 则应关闭 CoW.

如果一定要放在 HDD 上, 则应该在部署服务前先创建一个相应的空目录, 并设置在这个目录下设置禁用 CoW:

```shell
chattr +C /mnt/nas0/path/to/database
```

注意, `chattr +C` 只对空文件或者空目录生效. 如果目录里已有数据, 则应该先移开数据, 然后设置属性, 最后把数据移回来.

使用如下命令检查目录或文件是否是否启用了 CoW:

```shell
lsattr -d /path
lsattr filename
```

如果输出中包含一个 `C` 字母, 则说明 CoW 已经禁用了.

与前文相反, 满足以下特征时, 应保持开启 CoW:

- 媒体归档特征: 文件一旦写入就几乎永远不会再次修改内容, 只会读取. 如 Immich 的 `/upload` 目录, 存放所有照片的原始文件.
- 原子替换特征: 生成一个新版本的文件, 写入磁盘, 并删除旧文件. 比如办公文档, 图片处理等.

**2. 定期维护**:

- 清洗: btrfs 能检测数据损坏. `btrfs scrub start /mnt/nas0`.
- 均衡: 重写数据块, 回收未使用的空间. `btrfs balance start -dusage=5 -musage=5 /mnt/nas0`. 按需执行, 但是不要频繁做全量 balance, 建议每周或每月只处理使用率低于 5%~10% 的块. 除非磁盘爆满且无法写入, 否则**严禁**在机械硬盘上执行无参数的均衡操作, 因为那会重写整个磁盘, 耗时非常长, 且损耗非常大!

在删除了大量小文件时, 应该及时使用均衡命令来管理碎片化的可用空间.

**3. 碎片整理**:

这里的碎片指的是文件碎片. 由于 CoW 的特性, 对一个文件修改多次后, 它就会变成一连串的小的数据实体, 就像一块块碎片一样.

一定不要在挂载选项里开启 `autodefrag`, 这对机械硬盘的损伤是极大的. 不要盲目地启用碎片整理, 对 btrfs 而言, 只有在启用了 CoW 后, 对一个文件进行修改, 才会产生不连续的数据, 对于 NAS, 这种场景是非常小的, 因为 NAS 的数据存储通常是一次写入多次读取的.

| 场景特征 | 是否需要碎片整理 | 原因 |
| --- | --- | --- |
| 一次写入, 多次读取 | 不需要 | 常见的场景是将别处的大文件(如电影, 文档等)一次性复制到机械硬盘中, 写入时 btrfs 已经分配好了连续的空间, 生来就没有碎片 |
| 已有快照的文件 | 不需要 | 对快照进行碎片整理, 会破坏原有的快照数据块引用链接 |
| 长期追加的日志文件 | 需要 | 这是在长期修改的文件, 一段时间后就会变成若干个不连续的小碎块 |
| BT 下载目录 | 需要 | 如果直接用机械硬盘下载 BT, 由于多线程随机写入, 会导致文件破碎 |

所以不要在挂载硬盘时开启 `autodefrag` 选项, 而是应该每个月定期只对可能产生碎片的目录进行碎片整理:

```shell
# -t 32M: 只处理碎片化的, 大小小于 32M 的文件, 更大的不要去管
btrfs filesystem defragment -r -t 32M /mnt/nas0/fragments/dir
```

要检查是否开启了 `autodefrag` 选项, 只需要:

```shell
findmnt -t btrfs
```

### 远程访问

**⚠️注意**: 永远不要将浏览器文件管理器, 如 filebrowser 等服务, 挂载到机械硬盘上. 这种服务在启动时会计算目录大小等索引操作, 会让机械硬盘产生巨量的随机读取, 严重损耗硬盘寿命. 建议使用 sshfs/sftp 实现远程访问.

- PC 端: `sshfs orion@nas:/mnt/pool ~/NAS`
- 移动端: 开源的 [Material Files](https://github.com/zhanghai/MaterialFiles) 和闭源的 [Solid Explorer](https://play.google.com/store/apps/details?id=pl.solidexplorer2&hl=zh-CN) 都是好用的工具.

## 网络

我使用 OpenWrt 作为软路由, 本地服务器的主机名为 `aio.lan`. 要配置所有 `*.aio.lan` 的域名指向服务器, 只需要在 OpenWrt 的设置界面中找到 网络 -> DNS -> 常规 页面中的 **地址** 项.

其语法是: `/doman.example/ip-addr`, 会将所有匹配 `domain.example` 的域名解析到 `ip-addr` 上.

例如我的配置是: `/aio.lan/192.168.114.114`, 会将所有对 `*.aio.lan` 的域名解析到本地服务器, 以便 `nginx` 做反向代理.

## 应用服务

我部署的所有服务的模板已在 [Orion-zhen/self-hosted](https://github.com/Orion-zhen/self-hosted) 开源. 下面简要介绍两个服务.

### Filestash

Filestash 是一个浏览器文件管理器. 不同于 Filebrowser, 它不会对硬盘造成太重的读写负载. 一般来说, 推荐使用 SFTP 的方式从 Filestash 访问宿主机的文件系统. 示例 compose 文件如下:

```yaml
services:
  filestash:
    container_name: filestash
    image: machines/filestash:latest
    restart: unless-stopped
    volumes:
      - ./data:/app/data/state
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 5100:8334
    # 修复 host.docker.internal
    extra_hosts:
      - host.docker.internal:host-gateway
```

之后在 Filestash 界面中即可通过 SFTP, 从 `host.docker.internal` 域名访问宿主机的文件系统.

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
