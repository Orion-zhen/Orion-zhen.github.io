---
date: "2026-03-19T21:45:10+08:00"
draft: true
title: "笔记本上的 Arch Linux"
slug: "arch-laptop"
description: "续航和性能的综合调优"
image:
categories:
  - 教程
tags:
  - 续航
  - 笔记本
license: GNU General Public License v3.0
---

## 性能调优

一般地, Linux 桌面倾向于采用 power-profile-daemon (ppd) 来管理 CPU 调度策略, 但这个还是不够个性化. 能够给用户更精细的调度控制的 tlp 包提供了一个 ppd 等效的接口, 名为 tlp-pd. 使用如下命令安装相关的包:

```bash
sudo pacman -S tlp-pd tlp-rdw
```

安装完成后, 开启 tlp 相关服务:

```bash
sudo systemctl enable --now tlp.service tlp-pd.service
```

tlp 的可选配置项在 `/etc/tlp.conf` 中, 可以通过在 `/etc/tlp.d` 中添加自己的配置来覆盖默认配置.

## 续航调教

接下来我们着重优化笔记本在电池模式下的续航能力, 同时尽量保证离电状态下的使用体验. 考虑到我使用的是 Intel Ultra X7 358H 处理器, 可以安装 thermald 包来启用 Intel 官方的温控能力:

```bash
sudo pacman -S thermald
```

```bash
sudo systemctl enable --now thermald.service
```

接着来调教续航. 将以下内容保存为 `/etc/tlp.d/balance-power.conf`:

```ini
# 确保 pstate 驱动运行在 active 模式下, 硬件全权接管 pstate 控制.
# 这是发挥 Intel 芯片能效比的关键
CPU_DRIVER_OPMODE_ON_BAT=active
CPU_DRIVER_OPMODE_ON_SAV=active

# 使用 powersave 模式指导频率调度
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_SCALING_GOVERNOR_ON_SAV=powersave

# 限制频率上限百分比
CPU_MAX_PERF_ON_BAT=80
CPU_MAX_PERF_ON_SAV=60

# 禁止电池模式下 CPU 睿频
CPU_BOOST_ON_BAT=0
CPU_BOOST_ON_SAV=0

# 禁止动态加速特性, 不要在等待 IO 时提高 pstate
CPU_HWP_DYN_BOOST_ON_BAT=0
CPU_HWP_DYN_BOOST_ON_SAV=0

# 能量/性能策略. balance_power 是一个比较好的性能平衡点
CPU_ENERGY_PERF_POLICY_ON_BAT=balance_power
CPU_ENERGY_PERF_POLICY_ON_SAV=power
```

## 睡眠调教

使用 `systemd-logind` 来调整合盖行为. 一般推荐 `suspend-then-hibernate` 机制, 短时间内先睡眠, 长时间时休眠到硬盘. 编辑 `/etc/systemd/logind.conf`:

```ini
[Login]
HandleLidSwitch=suspend-then-hibernate # 默认合盖行为
HandleLidSwitchExternalPower=suspend # 电源接通时合盖则不会休眠
```

接下来配置 `systemd-sleep`. 编辑 `/etc/systemd/sleep.conf`:

```ini
[Sleep]
# 睡眠模式:
# s2idle: 唤醒极快, 类似手机息屏. 但略微耗电
# deep: 唤醒需1~2秒, 但更省电.
# 留空让内核自行决定
MemorySleepMode=
# 1小时后进入休眠
HibernateDelaySec=1h
#HibernateOnACPower=yes
```

最后重启电脑.

## 休眠配置

[参考 wiki](https://wiki.archlinuxcn.org/wiki/%E7%94%B5%E6%BA%90%E7%AE%A1%E7%90%86/%E6%8C%82%E8%B5%B7%E4%B8%8E%E4%BC%91%E7%9C%A0).

要启用 hibernate 机制, 需要在硬盘上有 swapfile. 仅使用 zram 无法实现休眠到硬盘.

创建一个 Btrfs 子卷作为 swapfile 地址:

```bash
sudo btrfs subvolume create /swap
```

配置 swapfile 路径的性质. 要求关闭压缩和 CoW 功能:

```bash
sudo chattr -c /swap
```

```bash
sudo chattr +C /swap
```

创建专门的 swapfile:

```bash
sudo btrfs filesystem mkswapfile --size 40G /swap/file
```

用于休眠的 swapfile 的大小需大于等于系统内存大小.

接着设置开机挂载, 以及交换优先级. 在存在 zram 的情况下, swapfile 应该只用于处理系统休眠, 防止频繁交换导致 SSD 寿命快速下降.

> 可以安装 zswap-disable-writeback 来禁止在非休眠时写入 swapfile.

在 `/etc/fstab` 末尾添加:

```text
# 交换文件, 设置 pri=0 以降低优先级
/swap/file  none  swap defaults,pri=0 0 0
```

使用 `sudo findmnt --verify` 验证配置, `sudo systemctl daemon-reload` 加载配置.

接着获取 swap 路径的 UUID 和 swapfile 的偏移量. 首先是 swap 路径的 UUID:

```bash
findmnt -no UUID -T /swap/file
```

然后是 swapfile 的偏移量:

```bash
sudo btrfs inspect-internal map-swapfile /swap/file
```

配置内核启动参数, 以便能找到 swapfile. 编辑 `/etc/default/grub`, 在 `GRUB_CMDLINE_LINUX_DEFAULT` 中添加:

```text
resume=UUID=<UUID> resume_offset=<offset>
```

最后是配置 mkinitcpio 的 resume 钩子, 否则唤醒时系统会无视硬盘里的休眠数据, 直接全新启动. 如果启用了 LVM, 则需要将 `resume` 钩子放在 `lvm2` 之后, 但要放在 `filesystems` 之前.

编辑 `/etc/mkinitcpio.conf`, 在 `lvm2` 之后, `filesystems` 之前加入 `resume` 模块.

然后重新生成内核的 initramfs 镜像:

```bash
sudo mkinitcpio -P
```

当一切完成后, 执行重启. 接着随便打开几个应用, 然后在命令行执行:

```bash
sudo systemctl hibernate
```

然后电源指示灯会闪烁一会儿(正在把内存写入 swapfile), 最后彻底断电关机. 当再次按下电源键开机后, 经过 grub 界面后系统将会直接恢复到休眠前的状态. 说明休眠配置成功.

## 自动锁屏

一般地, Wayland 窗口管理器在系统进入睡眠/休眠后不会自动锁屏. 换言之, 如果你忘记了手动锁屏, 那么别人打开笔记本盖子就能直接操作你的电脑.

可以使用[睡眠钩子](https://wiki.archlinuxcn.org/wiki/%E7%94%B5%E6%BA%90%E7%AE%A1%E7%90%86/%E6%8C%82%E8%B5%B7%E4%B8%8E%E4%BC%91%E7%9C%A0#%E7%9D%A1%E7%9C%A0%E9%92%A9%E5%AD%90)来配置自动锁屏. 以 `hyprlock` 为例, 编辑 `/etc/systemd/system/user-suspend@.service`:

```ini
[Unit]
Description=User suspend actions
Before=sleep.target

[Service]
User=%I
Type=forking
Environment=DISPLAY=:0
# ExecStartPre= -/usr/bin/pkill -u %u unison ; /usr/local/bin/music.sh stop
ExecStart=/usr/bin/hyprlock -c path/to/hyprlock.conf
ExecStartPost=/usr/bin/sleep 1 # 小睡一秒防止锁屏失败

[Install]
WantedBy=sleep.target
```

启用服务即可:

```bash
sudo systemctl enable --noe user-suspend@<username>.service
```

## 指纹识别

`pam` 是 Linux 的鉴权系统, 配置目录在 `/etc/pam.d` 中. 在所有配置文件中, `system-auth` 是最基础的配置, 一般所有别的鉴权配置都会引用这个配置. 在这里添加指纹识别模块, 可以让所有鉴权的地方, 如锁屏, polkit, 终端 `sudo` 提权等, 都可以刷指纹验证.

`pam` 的机制默认是串行的, 换言之, 只有一个验证项被通过/跳过了才能进入下一个验证. 换言之, 在等待验证指纹的时候不能输入密码.

虽然有 `pam-fprint-grosshack` 这样的 hack 技巧可以并发地读取指纹和密码, 但有安全隐患, 不推荐这么做.

要让指纹验证全局可用, 只需要编辑 `/etc/pam.d/system-auth`, 在 `pam_faillock.so` 所在行的后面一行加入:

```text
auth sufficient pam_fprintd.so
```

经测试, 在图形化界面中常常可以做到指纹和密码并行读取, 但在终端中则是串行的, 要跳过指纹读取, 必须按下 `Ctrl+C` 终止相关进程, 才会进入密码验证.

使用 `fprintd-enroll` 来录入指纹. 默认录入右手食指. 使用 `fprintd-verify` 来验证指纹, `fprintd-delete $USER` 来删除当前用户的所有指纹.

另外, gdm 对指纹+密码的验证模式支持得很好, 建议使用 gdm 作为登录器.

## 亮度配置

使用 clight 来自动调控屏幕背光. 从 AUR 安装 clight, 一并安装可以智能获取位置的 geoclue. 我的 clight 配置已经加入 `Orion-zhen/dot-config` 中.

geoclue 可以使用 IP 来获取位置. 为了防止梯子导致 IP 乱飞, 建议在 `/etc/geoclue/geoclue.conf` 中禁用 IP 定位.

启用所有的系统级服务:

```bash
sudo systemctl enable --now clightd.service geoclue.service
```

在 `~/systemd/user/` 中创建 `clight.service`:

```ini
[Unit]
Description=Clight - Lightweight screen brightness daemon
Documentation=https://github.com/FedeDP/Clight/wiki
# 确保在 D-Bus 启动后再运行，因为 clight 需要通过 D-Bus 与后台 clightd 通信
After=graphical-session.target dbus.socket
PartOf=graphical-session.target

[Service]
Type=dbus
BusName=org.clight.clight
ExecStart=/usr/bin/clight
Restart=on-failure
# 遇到意外崩溃时自动重启
Restart=on-failure
RestartSec=3

[Install]
WantedBy=graphical-session.target
```

启用用户级 clight 服务:

```bash
systemctl --user enable --now clight.service
```
