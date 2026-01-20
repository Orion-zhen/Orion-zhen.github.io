---
date: "2025-07-23T14:40:48+08:00"
draft: false
title: "Windows 食用指南"
slug: "live-with-windows"
description: "相对优雅地使用 Windows"
image: /post/tutorial/live-with-windows/cover.jpg
categories:
  - 教程
tags:
  - 环境配置
  - Windows
license: GNU General Public License v3.0
---

> [封面出处](https://www.pixiv.net/artworks/123948772)

当你因为各种原因而不得不使用 Windows 时, 本教程将提供一份求生指南.

## 安装

Windows 非常神必的一点在于, 使用默认镜像安装后, 会要求联网并登录微软账号. 在登陆账号后, 电脑的默认账号和 `$HOME` 目录名都会是微软账号邮箱的前 5 位. 而且后续重命名非常麻烦, 甚至要将以前安装的应用卸载重装. 这里推荐从 Windows 官方下载安装镜像后, 使用 [Rufus](https://rufus.ie/zh) 将镜像写入到 U 盘中. 写入后 Rufus 会弹出配置选项对话框, 在这里可以选择指定本地用户名, 以及跳过联网等操作, 非常方便.

## 包管理器

Windows 下没有像 Arch Linux 那么强大的包管理器. 微软官方提供了一个名为 [winget](https://learn.microsoft.com/zh-cn/windows/package-manager) 的包管理工具. 但本质上也就是个自动化安装脚本, 它做的事情就只是从源中配置的下载地址下载安装包, 然后自动安装罢了. 至于安装到哪里, 就不是 winget 能决定的了. 但无论如何, winget 终归还是提供了一个类似 `pacman` 的使用体验, 而且还是微软原生的.

要搜索软件, 可以:

```powershell
winget search "app key words"
```

注意, 多个单词需要用 `"` 包裹起来.

安装软件:

```powershell
winget install <software.id>
```

软件 ID 在搜索结果中会展示.

更新软件:

```powershell
winget update --all --include-unknown
```

卸载软件:

```powershell
winget remove <software.id>
```

winget 还提供了导出和导入配置的功能, 便于在新系统中快速安装需要的应用.

导出配置:

```powershell
winget configure export --all -o <file.yaml>
```

导入配置:

```powershell
winget configure -f <file.yaml>
```

winget 的源只包含软件的元数据, 即软件名, 版本, 下载地址等. 实际下载软件还是要访问软件自己的下载链接. 所以给 winget 换源其实意义不大.

对于不在 winget 中的软件, 我的方案是统一安装到一个指定的目录下, 例如 `C:/Applications`.

> 这样其实挺别扭的, winget 把软件装在哪里我无法控制, 而我自己又显式地指定了手动安装的软件的位置. 一个系统下两套软件管理方案并存, 不是很优雅.

## 字体

Windows 下没法从 winget 直接安装字体, 需要从网上下载字体包, 解压后选中字体, 在右键菜单中手动安装. 我常用的字体是 [Maple Mono NF](https://github.com/subframe7536/maple-font/releases/latest) 和 [Hack Nerd Font](https://www.programmingfonts.org/#hack).

## 配置文件

一些跨平台的工具, 如 `git`, 在 Windows 下依然支持 `$HOME/.config` 的配置文件形式. 所以可以将 Linux 下的 `.config` 文件夹整个克隆到 Windows 的 `$HOME` 目录下.

> 哦不对, 在 Windows 下应该是 `%USERPROFILE%/.config`

## Terminal & PowerShell

Terminal 是 Windows 内置的终端模拟器, 也是最推荐用的. PowerShell 则是 Windows 下的 Shell.

> 哭了, WezTerm 在 Windows 下背景透明有问题, 呜呜我的跨平台方案.

首先配置 Terminal. 在 设置 -> 默认值 -> 外观 中, 可以对 Terminal 的外观进行若干更改. 我习惯的方案如下:

- 配色方案: Dark+
- 字体: Maple Mono NF
- 字号: 14
- 光标形状: 实心框
- 背景不透明度: 70%
- 亚克力材料: 启用

接着是 PowerShell 的配置. 值得注意的是, Windows 下常存在两个 PowerShell. 一个是 PowerShell 5.x, 一个是最新版的 PowerShell. 可以通过 winget 安装最新版的 PowerShell:

```powershell
winget install microsoft.powershell
```

二者在 Terminal 中的区别是, PowerShell 5.x 的配置文件名为 "Windows PowerShell", 而最新版的 PowerShell 的配置文件名为 "PowerShell", 需要注意区分. 为了避免混乱, 可以在 设置 -> 启动 -> 默认配置文件 中选择 "PowerShell".

与 Linux 下不同, PowerShell 的配置文件常是

```powershell
C:/Users/<username>/Ducoments/PowerShell/Microsoft.PowerShell_profile.ps1
```

或者通过环境变量 `$PROFILE` 访问.

在使用自定义配置之前, 需要允许 PowerShell 加载自定义脚本. 打开管理员终端, 然后输入:

```powershell
Set-ExecutionPolicy Bypass
```

安装需要的模块:

```powershell
Install-Module posh-git
Install-Module CompletionPredictor
```

以下是一份简单的配置文件:

```powershell
### 初始化 ###
Import-Module PSReadLine
Import-Module posh-git
Import-Module CompletionPredictor
# 初始化 starship
if (Get-Command starship -ErrorAction SilentlyContinue) {
    Invoke-Expression (&starship init powershell)
}
# 初始化 zoxide
if (Get-Command zoxide -ErrorAction SilentlyContinue) {
    Invoke-Expression (& { (zoxide init powershell | Out-String) })
}

### 自定义函数 ###
function ListDirectories {
    # 检查 'eza' 命令是否存在
    try {
        $eza_exists = Get-Command eza -ErrorAction SilentlyContinue
    }
    catch {
        $eza_exists = $null
    }

    if ($null -ne $eza_exists) {
        # eza 存在, 构建 eza 的参数列表
        $eza_params = @(
            '--icons',
            '--group-directories-first',
            '--color=always',
            '--header',
            '--long',
            '--git',
            '--no-user',
            '--no-permissions',
            '--no-time'
        )

        # 将用户输入的原始参数 ($args) 添加到 eza 参数列表后面
        $all_params = $eza_params + $args

        # 执行 eza 命令
        & eza @all_params
    }
    else {
        # eza 不存在, 调用 PowerShell 原生的 Get-ChildItem 命令
        Get-ChildItem @args
    }
}

function ListAllItems {
    # 检查 'eza' 命令是否存在
    try {
        $eza_exists = Get-Command eza -ErrorAction SilentlyContinue
    }
    catch {
        $eza_exists = $null
    }

    if ($null -ne $eza_exists) {
        # eza 存在, 构建 eza 的参数列表
        $eza_params = @(
            '--icons',
            '--group-directories-first',
            '--color=always',
            '--header',
            '--long',
            '--git',
            '--all'
        )

        # 将用户输入的原始参数 ($args) 添加到 eza 参数列表后面
        $all_params = $eza_params + $args

        # 执行 eza 命令
        & eza @all_params
    }
    else {
        # eza 不存在, 调用 PowerShell 原生的 Get-ChildItem 命令
        Get-ChildItem @args
    }
}

### 配置自动补全 ###
# 启用历史记录和插件
Set-PSReadLineOption -PredictionSource HistoryAndPlugin
# 列表显示补全项
Set-PSReadLineOption -PredictionViewStyle ListView

### 自定义命令 ###
# 移除原有 ls
Remove-Item -Path alias:ls -Force -ErrorAction SilentlyContinue
New-Alias -Name ls -Value ListDirectories -Scope Global -Description "eza-powered ls"
New-Alias -Name la -Value ListAllItems -Scope Global -Description "eza-powered ls -a"

if (Get-Command fastfetch -ErrorAction SilentlyContinue) {
    New-Alias -Name ff -Value fastfetch -Scope Global
}

if (Get-Command bat -ErrorAction SilentlyContinue) {
    Remove-Item -Path alias:cat -Force -ErrorAction SilentlyContinue
    New-Alias -Name cat -Value bat -Scope Global -Description "better cat"
}

if (Get-Command zoxide -ErrorAction SilentlyContinue) {
    Remove-Item -Path alias:cd -Force -ErrorAction SilentlyContinue
    New-Alias -Name cd -Value z -Scope Global -Description "modern cd by zoxide"
}

### 键位设置 ###
# Crtl+D 退出终端
Set-PSReadlineKeyHandler -Key "Ctrl+d" -Function ViExit
# Ctrl+Z 撤销
Set-PSReadLineKeyHandler -Key "Ctrl+z" -Function Undo
```

## 激活

打开**最新版**的 PowerShell, 输入:

```powershell
irm https://get.activated.win | iex
```

然后根据提示操作即可.

## WSL2

### 安装

首先在系统 设置 -> 系统 -> 可选功能 -> 更多 Windows 功能(在页面最下方) 中勾选 Hyper-V 和适用于 Linux 的 Windows 子系统两项. 最新版本的 Windows 会自动安装 Linux 6.6 LTS 内核.

在命令行中输入 `wsl` 即可查看相关命令. Arch Linux 已经作为官方维护的 Linux 发行版被加入到 WSL2 中. 这里我选择安装 Arch Linux 为我的 WSL2 发行版系统.

```powershell
wsl --install archlinux
```

### 配置用户

安装好 Arch Linux WSL2 后, 将会自动进入 WSL2 命令行. 此时默认登录用户是 root, 没有密码. 这是非常不推荐的行为, 甚至诸如 `yay` 等命令会禁止在 root 下运行. 所以需要创建一个非 root 用户并授予其 root 权限, 最后将这个用户设置为默认登录用户.

首先更新并安装基本软件:

```bash
pacman -Syu
```

```bash
pacman -S base-devel vim sudo
```

接着创建新用户, 使用 `-m` 参数创建对应的 `$HOME` 目录:

```bash
useradd -m <username>
```

为新创建的用户设置密码:

```bash
passwd <username>
```

将新创建的用户加入到 sudo 组中. 在 Arch Linux 下, sudo 组名为 `wheel`:

```bash
usermod -aG wheel <username>
```

接着授予 `wheel` 组 sudo 权限. 编辑 `/etc/sudoers` 文件, 找到

```text
# %wheel ALL=(ALL:ALL) ALL
```

这一行, 将其取消注释.

最后, 设置新创建的用户为默认登录用户. 找到 `/etc/wsl.conf`, 向其末尾添加以下内容:

```toml
[user]
default = <username>
```

接着重启 WSL, 即可生效.

### 配置网络

打开 WSL Settings 应用, 找到网络栏目. 网络模式建议选择 `Mirrored`, 这样可以充分利用 WSL 和 Windows 深度集成的优势. 但要注意, 如果在 clash 类软件中打开了 tun 模式, 则需要将 tun 模式堆栈切换为 System 模式, 否则 WSL 将无法联网.

还可以开启主机地址环回, 这样就可以通过 `127.0.0.1` 来访问 WSL 内的网络服务.

### 配置中文

编辑 `/etc/locale.gen` 文件, 找到

```text
#zh_CN.UTF-8 UTF-8
```

这一行并取消注释. 保存, 然后运行:

```bash
locale-gen
```

### 配置内核

微软给 WSL2 配置的默认内核是 Linux 6.6, 而上一个版本是 5.19. 足以见得微软对内核更新非常不上心. 我写了一个 GitHub Actions, 会自动拉取最新的 Linux 内核, 每周构建. 你可以在[这里](https://github.com/Orion-zhen/wsl-kernel-build/releases)找到最新构建的内核映像.

在下载完 `bzImage` 映像后, 将其放在一个你认为妥当的地方, 然后打开 WSL Settings -> 开发商 -> 自定义内核, 选中你下载的内核映像, 然后重启 WSL2 即可.

你可以通过 `uname -r` 命令来检查自己的内核版本.

## 双系统

> 巨坑无比, 我会因此记恨 Windows 一辈子.

总体而言, 双系统有两种方案, 装在同一个硬盘上和装在不同硬盘上.

对于装在同一个硬盘上的情况, 我十分不推荐. 因为 Windows 会弄乱 Linux 的启动分区, 常会导致二者都无法启动. 如果一定要装双系统的话, 我建议装在两块不同的硬盘上.

然而即使是装在两块不同的硬盘上, 在安装时也有严格的先后顺序要求. 需要指出的是, Windows 的安装逻辑是:

1. 扫描电脑中所有的硬盘.
2. 如果发现在某块硬盘上已经有一个 EFI 系统分区(ESP), 则优先使用这个分区来存放自己的引导文件.
3. 当且仅当所有的硬盘上都找不到可用的 ESP 时, Windows 才会选择在安装过程中指定的硬盘创建自己的 ESP.

这带来的问题是: 在安装 Windows & Linux 双系统时, 必须严格遵循**先 Windows 后 Linux** 的顺序. 否则 Windows 将变得非常不稳定. 原因在于, 引导配置数据(BCD)与系统状态不一致. 在 Windows 运行过程中, 尤其是在系统更新, 驱动安装, 休眠唤醒等操作中, 会更新系统 BCD 信息. 而 Windows 无法正确, 稳定地跨物理磁盘操作而导致 BCD 文件损坏或不一致. 一个损坏的 BCD 会在下次启动时传递错误的参数给内核, 而导致启动失败.

具体的表现情况可能是: 在没有按顺序装好 Windows & Linux 双系统后, 可能一段时间内 Windows 可以正常运行, 但会在随机的某个时刻发生崩溃蓝屏, 错误代码一般为 `CRITICAL_PROCESS_DIED`, 且一旦蓝屏出现, 则不可恢复.

如果想要在一个已经装好双系统的电脑上仅重装 Windows, 则需要将 Linux 隔离开. 可以物理拆卸 Linux 硬盘, 或者在 BIOS 中禁用对应的硬盘, 或者干脆将 Linux 一道重装. 也可以尝试暂时抹去 Linux ESP 的标记, 让 Windows 无法识别到 Linux 的 ESP, 从而正确地创建自己的 ESP, 待 Windows 安装完成后再恢复 Linux ESP 的标记.

[GParted](https://gparted.org/) 或许可以做到.

1. 下载 [GParted ISO 镜像文件](https://gparted.org/download.php).
2. 从这个镜像文件启动, 找到 Linux 所在的硬盘中的 EFI System Partition, 文件系统类型通常为 FAT32.
3. 右键那个分区, 选择管理标记(Manage Flags).
4. 在标记列表中, 取消勾选 `esp` 标记.
5. 保存, 应用所有操作, 退出.

待 Windows 完成后, 进行上述操作的逆操作即可.

在安装完双系统后, 会有时间不同步的问题. 这是因为 Windows 会将主板时间解释为 RTC, 而 Linux 会将主板时间解释为 UTC. 为了解决这个问题, 最方便的是在 Linux 上修改, 使其将主板时间解释为 UTC:

```bash
timedatectl set-local-rtc 1 --adjust-system-clock
```

但是将 Linux 的时间配置更改后会导致 Linux NTP 失效等问题, 所以还是建议修改 Windos 的注册表, 将主板时间解释为 UTC. 打开管理员终端, 执行:

```pwsh
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f
```

然后重启即可应用改变.
