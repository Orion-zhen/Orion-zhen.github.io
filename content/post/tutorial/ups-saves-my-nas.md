---
date: "2026-01-28T18:13:45+08:00"
draft: false
title: "UPS Linux 使用教程"
slug: "ups-saves-my-nas"
description: "UPS 拯救 NAS!"
image:
categories:
    - 教程
tags:
    - UPS
license: GNU General Public License v3.0
---

本教程将讲述如何使用 UPS 连接 NAS, 并在断电时自动保存数据并安全关机.

## 安装软件

Debian Linux 下安装以下软件:

```bash
sudo apt install nut nut-monitor nut-cgi nut-i2c nut-modbus nut-xml nut-snmp
```

## 识别 UPS 并配置驱动

首先将 UPS 的 USB 线接入电脑, 然后执行 `lsusb` 检查是否可以认到 UPS.

如果可以, 则进入 `/etc/nut/ups.conf` 配置对这个 UPS 的驱动. 在文件末尾添加:

```toml
[myups]
    driver = usbhid-ups
    port = auto
    desc = "My UPS"
```

其中 `[myups]` 是 UPS 的名称, 可以自定义.

## 配置 nut 服务模式

在 `/etc/nut/nut.conf` 中, 将 `MODE` 改为 `standalone`. 即可对单台接入 UPS 的设备进行配置.

## 配置监控和守护进程

需要配置 upsd (服务端) 和 upsmon (客户端，负责执行关机指令). 即使是单机, NUT 也是通过内部网络协议通信的.

### 1. 配置用户

为了让监控进程能访问服务, 需要创建一个用户. 在 `/etc/nut/upsd.user` 中新增用户:

```toml
[upsmonitor]
    password = password
    upsmon primary
```

其中, `[upsmonitor]` 是用户名, `upsmon` 是监控进程的名称, `primary` 是监控进程的优先级.

### 2. 配置监控逻辑

在 `/etc/nut/upsmon.conf` 中, 找到 `MONITOR` 行, 修改或添加如下:

```toml
MONITOR myups@localhost 1 upsmonitor password primary
```

意为: 当 `myups` 断电时, 执行 `upsmon` 的监控逻辑.

## 进阶: 安全关机脚本

有时候需要等机械硬盘磁头归位, 所以不能直接关机了事. 创建 `ups-safe-fuckoff.sh` 文件:

```bash
#!/bin/bash

# ========== 配置区域 ==========
# 1. 挂载点
MERGERFS_POINT="/mnt/pool"
MOUNT_POINTS="/mnt/nas0 /mnt/nas1"
# 2. 需要停止的服务
SERVICES="docker.socket docker.service"
# 硬盘 ID 路径
DISKS="/dev/disk/by-id/ID1 /dev/disk/by-id/ID2"
LOG_FILE="/var/log/ups-fuckoff.log"
# ========== 结束配置 ==========

# ========== 安全关机 ==========

echo "$(date): UPS Alert! Starting safe fuckoff." >> $LOG_FILE

# 1. 停止上层服务
echo "Stopping services..." >> $LOG_FILE
systemctl stop $SERVICES

# 2. 同步数据到硬盘
echo "Syncing data..." >> $LOG_FILE
sync

# 3. 杀死占用硬盘的进程
echo "Killing processes using HDDs..." >> $LOG_FILE
ALL_MOUNTS="$MERGERFS_POINT $MOUNT_POINTS"
for mp in $ALL_MOUNTS; do
    if mountpoint -q $mp; then
        fuser -v -k -m $mp >> $LOG_FILE 2>&1
        echo "Killed process on $mp"
    fi
done
sync

# 4. 卸载 mergerfs
echo "Unmounting mergerfs..." >> $LOG_FILE
umount -f $MERGERFS_POINT || echo "Failed to umount $MERGERFS_POINT" >> $LOG_FILE

# 5. 卸载硬盘
for mp in $MOUNT_POINTS; do
    if mountpoint -q $mp; then
        umount -f $mp
        if [ $? -eq 0 ]; then
            echo "Umounted $mp" >> $LOG_FILE
        else
            echo "CRITICAL: Failed to umount $mp! Trying lazy umount..."
            umount -l $mp
        fi
    fi
done

# 6. 硬盘停转
echo "Stopping HDDs..." >> $LOG_FILE
for hdd in $DISKS; do
    hdparm -Y $hdd
done
sleep 5 # 给硬盘一点时间完成动作

# 6. 执行系统关机
echo "System is now fucked off!" >> $LOG_FILE
/sbin/shutdown -h now
```

授予可执行权限: `shmod +x ups-safe-fuckoff.sh`.

接着配置 `upsmon` 使用该脚本作为关机指令. 编辑 `/etc/nut/upsmon.conf`, 找到 `SHUTDOWNCMD`, 改为:

```toml
SHUTDOWNCMD "/path/to/ups-safe-fuckoff.sh"
```

验证关机流程:

```bash
sudo upsmon -c fsd
```

## 进阶: 关机触发时机

默认情况下, nut 会等待 UPS 电量低之后才触发关机指令. 但是为了最大化利用 UPS 电池, 推荐在市电异常后一段时间内就关机. 这时需要引入 `upssched`. 我们需要告知 `upsmon`, 当 UPS 状态发生变化时, 不要只是写日志, 而要把事件交给 `upssched` 处理.

首先编辑 `/etc/nut/upsmon.conf`, 修改或添加:

```toml
# 指定调度器程序位置
NOTIFYCMD /usr/sbin/upssched

# 定义哪些事件触发调度器(EXEC)
# ONBATT: 市电中断，切换到电池
# ONLINE: 市电恢复
NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC
NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
```

接着配置调度器逻辑, 编辑 `/etc/nut/upssched.conf`, 添加:

```toml
# 调度器执行的具体脚本路径
CMDSCRIPT /usr/bin/upssched-cmd

# 定义管道文件路径
PIPEFN /run/nut/upssched.pipe
LOCKFN /run/nut/upssched.lock

# ============ 核心逻辑 ============

# 场景 1: 检测到断电 (ONBATT)
# 动作: 启动一个名为 shutdown_timer 的计时器, 时长 60 秒
AT ONBATT * START-TIMER shutdown_timer 60

# 场景 2: 检测到市电恢复 (ONLINE)
# 动作: 如果计时器还在跑, 说明电来了, 取消关机任务
AT ONLINE * CANCEL-TIMER shutdown_timer
```

最后编辑调度脚本 `/usr/bin/upssched-cmd`, 添加:

```bash
#!/bin/bash
case $1 in
    shutdown_timer)
        # 计时器跑完了，说明市电还没恢复
        # 触发 FSD (Forced Shutdown)，这将调用在 upsmon.conf 里配置好的安全关机脚本
        logger -t upssched-cmd "Power is gone for too long. Triggering Safe Shutdown..."
        /usr/sbin/upsmon -c fsd
        ;;
    *)
        logger -t upssched-cmd "Unrecognized command: $1"
        ;;
esac
```

## 启动

当一切配置完成, 运行:

```bash
sudo systemctl enable --now nut.target
```

即可享受 UPS 带来的保障.
