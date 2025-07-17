---
date: "2025-01-10T19:49:43+08:00"
draft: false
title: "如何在 Windows 上优雅地编程"
slug: "how-to-code-on-windows"
description: "虽然但是, 我还是推荐你用 Arch Linux"
image: /post/tutorial/how-to-code-on-windows/cover.jpg
categories:
  - 教程
tags:
  - 环境配置
  - Windows
license: GNU General Public License v3.0
---

> [封面出处](https://www.pixiv.net/artworks/123948772)

本教程旨在让你在 Windows 平台上搭建一个优雅的开发环境. 众所周知, Windows 下除了使用 IDE, 代码开发, 尤其是 C/C++ 开发的体验较差. 而身为一名~~极客~~, 怎么能老老实实地使用 IDE 呢? 下面将给出我的配置方案以供参考

**前置条件**:

1. 安装了 Windows 11 (专业版/企业版/专业工作站版什么的都好, 但不要是家庭版) 的电脑. 如果你不会激活 Windows, 可以在文末找到激活方法
2. 安装了 [Visual Studio Code](https://code.visualstudio.com/download) (以下简称 VSCode/vsc)
3. 安装了新版的 [Terminal](https://apps.microsoft.com/detail/9n0dx20hk701?hl=zh-CN&gl=CN)

我的配置主要是用 WSL2 提供的 Linux 环境进行代码开发, 通过 VSCode 的 Remote-WSL 插件连接到 WSL2 环境进行代码编辑和运行等操作

## 安装配置 WSL2

### 启用 WSL2

打开开始菜单, 在搜索框中输入 **control**, 在搜索结果中找到**控制面板**并打开. 在**程序**菜单中找到**启用或关闭 Windows 功能**, 然后勾选以下选项:

- Hyper-V
- 适用于 Linux 的 Windows 子系统
- 虚拟机平台

![enable-wsl2](/post/tutorial/how-to-code-on-windows/enable-wsl2.png)

然后单击确定键, 等待系统完成操作, 然后**重启电脑**

### 安装 WSL2

打开 Terminal, 输入命令:

```shell
wsl --install
```

这将在你的电脑上安装 WSL2 软件. 等待命令完成, 然后运行以下命令将 WSL 的版本默认设置为 2:

```shell
wsl --set-default-version 2
```

### 安装 WSL2 Linux 内核

点击[这里下载内核安装包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi), 下载完成后双击打开, 然后按照提示安装即可

### 选择 Linux 发行版

你可以在 Microsoft Store 中搜索并安装你喜欢的 Linux 发行版. 这里我强烈推荐使用 Arch Linux, 可以搭配[我的 Arch Linux 配置方案食用](https://github.com/Orion-zhen/dotfiles)

> Terminal + WSL2 的环境下, starship 会有一点小 bug, 所以我换回了 p10k 主题

在发行版下载安装完成后, 打开 Terminal, 输入命令 `wsl`, 首次进入发行版会提示你创建用户名和密码, 完成初始化后即正式进入 WSL2 环境

在 WSL2 终端中输入 `code`, 等待命令执行完成即可在 WSL2 中打开 VSCode. 此时的 VSCode 通过 Remote-WSL 插件远程连接到 WSL2 环境, 你可以像在本地编辑器一样编辑代码, 也可以通过终端运行编译等操作. WSL2 文件系统会自动挂在到 Windows 文件系统上, 你可以直接在 Windows 文件资源管理器中找到 WSL2 文件夹进行文件管理. 在 WSL2 中, Windows 的文件系统被挂载在 `/mnt` 目录下, 例如 C 盘根目录的挂载路径为 `/mnt/c`

### 使用配置文件配置 WSL2

参考[微软官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config#wslconfig), 在 Windows Terminal 中输入命令:

```shell
code %USERPROFILE%\.wslconfig
```

这里给出我自己的配置文件:

```text
[wsl2]
memory=30GB
networkingMode=NAT
dnsTunneling=true
autoProxy=true

[experimental]
autoMemoryReclaim=gradual
bestEffortDnsParsing=true
```

完成配置后, 关闭 WSL2:

```shell
wsl --shutdown
```

重新启动 WSL2 后, 配置即可生效

### (可选)升级 WSL2 内核

打开 WSL2 终端, 输入命令 `uname -r`, 你会发现当前的内核版本是 `5.x`, 远远落后于当前的最新 Linux 内核版本. 虽然不是太影响正常使用, 但看着就很不舒服. 身为~~极客~~, 怎么能忍住不升级呢?

首先安装依赖. 此处假设你在使用 Arch Linux, 其它发行版请自行替换对应的软件包名:

```shell
sudo pacman -Syu base-devel flex bison pahole openssl libelf bc wget python
```

然后获取新版的 Linux 内核包并解压:

```shell
wget -O- https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-<Your-Kernel-Version>.tar.xz | tar -xJf-
```

将 `<Your-Kernel-Version>` 替换为你当前的内核版本号. 例如在本文撰写时, 最新的稳定版为 `6.12.9`, 于是相应的命令为:

```shell
wget -O- https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.9.tar.xz | tar -xJf-
```

进入解压后的目录:

```shell
cd linux-<Your-Kernel-Version>
```

下载微软给出的 WSL2 内核编译配置:

```shell
mkdir Microsoft
wget -O Microsoft/config-wsl https://github.com/microsoft/WSL2-Linux-Kernel/raw/linux-msft-wsl-6.1.y/arch/x86/configs/config-wsl
```

使用 VSCode 打开微软的配置:

```shell
code Microsoft/config-wsl
```

找到 `CONFIG_LOCALVERSION` 所在的那一行, 你可以将这个值改为你喜欢的样子, 例如我就改成了:

```text
CONFIG_LOCALVERSION="-Orion-WSL2"
```

保存并退出, 然后回到 Linux 内核目录, 使用如下命令编译内核:

```shell
yes "\n" | make -j8 KCONFIG_CONFIG=Microsoft/config-wsl
```

`-j8` 表示使用 8 个并行任务进行编译, 你可以根据你实际的 CPU 核心数量进行调整. 注意, 对于 Intel 12 代及以后的 CPU, 请不要使用超过大核数量的并行任务, 因为大小核异构会导致编译失败. 编译完成后, 会出现如下字样:

```text
Kernel: arch/x86/boot/bzImage is ready
```

此时内核编译成功, 将编译成功的内核复制到 Windows 的文件系统下:

```shell
cp arch/x86/boot/bzImage /mnt/<Your-Path>
```

在 `.wslconfig` 文件中的 `[wsl2]` 域内加入如下字段:

```text
[wsl2]
kernel=<Your-Path>
```

其中, `<Your-Path>` 是 Windows 文件系统中内核文件的**绝对路径**. 完成后, 重启 WSL2 即可生效. 再次输入命令 `uname -r` 即可查看到新版的内核版本号:

![linux-6.12.9-Orion-WSL2](/post/tutorial/how-to-code-on-windows/linux-6.12.9-Orion-WSL2.png)

## VSCode 插件推荐

除了 VSCode 本身, 我还推荐安装以下 VSCode 插件以提高开发效率:

- Auto-Save on Window Change: 随着窗口切换自动保存文件
- autoDocstring - Python Docstring Generator: 自动生成 Docstring 风格的 Python 注释
- Better C++ Syntax: 更好的 C++ 语法高亮
- Better Comments: 代码注释更加美观
- Black Fomatter: Python 代码格式化工具
- Bookmarks: 给代码中的重要位置加书签, 以便快速追踪
- LaTeX Workshop: 在 VSCode 中编写和预览 LaTeX 文档
- Markdown All in One: 编写 Markdown 文档更加方便
- Markdown PDF: 将 Markdown 文档导出为 PDF 格式
- Markdown Preview Enhanced: 更好地预览 Markdown 文档
- Markdownlint: Markdown 语法检查
- Output Colorizer: 输出日志文件的颜色高亮
- parquet-viewer: 预览 Parquet 文件
- Path Autocomplete: 自动补全文件路径

至此, 一个优雅的 Windows 开发环境就初步搭建完成了. 祝你在 Windows 平台上编程愉快!

## 激活 Windows

关闭你的所有杀毒软件, 然后在 **PowerShell** 中输入以下命令:

```powershell
irm https://get.activated.win | iex
```

然后根据提示操作即可


