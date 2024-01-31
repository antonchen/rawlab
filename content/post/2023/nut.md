---
title: "NUT 配置山特 MT1000S"
slug: ""
date: 2023-03-08T14:18:15+08:00
draft: false
tags: ["NUT", "UPS", "HomeLab", "RS232"]
categories: ["Linux"]
author: "Anton Chen"
comment: true
toc: true
---

用了五六年的二手 APC BK650-CH 坏了，市电断电不切换，找人维修未果。而后在小黄鱼买了山特 MT1000S，这是一款后备式长效机，外接 24V 电池使用，600W、RS232，加上运费 163，电池用电瓶车上拆下的 20AHx4 2串2并。

本来提前在淘宝上买了一根比较好的 COM 线，结果用的时候发现买错了，在小县城转了一圈没买到成品线，只找到一家卖 DB9 插头的，然而奸商要价太高且买不到 4x0.5 的电缆，放弃了。最后从淘宝下单了镀金的 DB9，同时买了一根一米的 4x0.5 带屏蔽的电缆，总计花费 6.9 元。

记录一下 NUT 配置。
<!--more-->

## 安装

Debian 上直接安装 `nut` 假包就行了，如果非单机接入 UPS，只需要在其它机器安装 `nut-client` 即可。

## 配置

### 驱动配置

山特 MT 系列是使用的 `Megatec/Q1` 协议变种，可以使用 `blazer_*` 和 `nutdrv_qx` 驱动，其中 `blazer_ser` 是 RS232 驱动，USB 接口则是 `blazer_usb`，而 `nutdrv_qx` 则不区分。但是测试发觉 `nutdrv_qx` 是对电池最高电压与最低电压和当前电压来计算电池电量百分比，与 `blazer_*` 驱动没有区别，同样[不支持电池电压设置为低电压](https://github.com/networkupstools/nut/issues/561)，同时 `nutdrv_qx` 启动速度似乎比较慢，所以我选择使用 `blazer_ser`，而 `nutdrv_qx` 相关配置注释保留。

`/etc/nut/ups.conf`

```conf
maxretry = 3
# 这个字段相当于一个用户名，之后的一系列操作都使用它
[ups]
    driver = blazer_ser
    # COM 口
    port = "/dev/ttyS0"
    # 描述信息
    desc = "MT1000S"
    # 来电开机等待时间（分钟）
    ondelay = 1
    # 停电时 FSD 关机等待时间（秒），即 UPS 延迟关机时间
    offdelay = 300
    # 忽略 UPS 自身低电量告警
    ignorelb
    # 剩余供电时间，-1 不启用
    override.battery.runtime.low = -1
    # 剩余电量低于 45% 即为低电压
    override.battery.charge.low = 45
    # 电池最高电压
    default.battery.voltage.high = 27.5
    # 电池最低电压
    default.battery.voltage.low = 22
    # 电池标称电压
    default.battery.voltage.nominal = 24
#     driver = "nutdrv_qx"
#     protocol = "Q1"
#     port = "/dev/ttyS0"
#     desc = "MT1000S"
#     ondelay = 60
#     offdelay = 300
#     pollfreq = 1
#     override.battery.charge.low = 70
#     default.battery.voltage.high = 27.5
#     default.battery.voltage.low = 22
#     default.battery.voltage.nominal = 24
```

配置文件保存后驱动会自动重启，查看驱动状态

```
systemctl status nut-driver@ups.service
```

NUT 默认是以普通用户 `nut` 运行的，需要对这个用户赋予串口或 USB 接口权限，否则驱动启动不成功。对于串口只需要添加用户到 `dialout` 组即可，USB 口请自行查阅。

```shell
usermod -a -G dialout nut
```

### 监听端口

`/etc/nut/upsd.conf`

```conf
LISTEN 0.0.0.0 3493
```

### 模式

分别可以设置 `standalone` 单机,`netserver` 服务端，`netclient` 客户端。

`/etc/nut/nut.conf`

```conf
MODE=netserver
```

### 用户设置

`/etc/nut/upsd.users`

```conf
[admin]
    password = masterpassword
    instcmds = ALL
    actions = FSD
    actions = SET
    upsmon master

[slave]
    password  = slavepassword
    upsmon slave
```

### 重启服务端

```
systemctl restart nut-server.service
```

### 查看 UPS 状态

```
upsc ups
```

也可以使用 `watch -n 1` 一秒打印一次信息

```
watch -n 1 upsc ups
```

### 监控配置

此处只提供服务端配置，客户端自行修改相关参数。需要说明的是市电断电关机需要通过 `upsmon -c fsd` 关机，这样 UPS 才会自动关机，从而实现主板来电自启。

>当 UPS 触发低电量时，监控会自动执行 `upsmon -c fsd` 关机。

`/etc/nut/upsmon.conf`

```conf
RUN_AS_USER root
# 服务器端配置
MONITOR ups@localhost 1 admin masterpassword master
# FSD 调用关机命令
SHUTDOWNCMD "/sbin/shutdown -h +0"
# 通知程序
NOTIFYCMD /sbin/upssched
POLLFREQ 5
POLLFREQALERT 3
HOSTSYNC 15
DEADTIME 15
# 创建一个文件供脚本判断
POWERDOWNFLAG /etc/killpower
FINALDELAY 5

# 市电恢复
NOTIFYMSG ONLINE    "UPS %s on line power"
# 市电断开
NOTIFYMSG ONBATT    "UPS %s on battery"
# 电池电量低
NOTIFYMSG LOWBATT   "UPS %s battery is low"
# FSD 关机中
NOTIFYMSG FSD       "UPS %s: forced shutdown in progress"
# UPS 连接断开
NOTIFYMSG COMMOK    "Communications with UPS %s established"
# UPS 连接恢复
NOTIFYMSG COMMBAD   "Communications with UPS %s lost"
# 关机
NOTIFYMSG SHUTDOWN  "Auto logout and shutdown proceeding"
# 电池失效
NOTIFYMSG REPLBATT  "UPS %s battery needs to be replaced"
# UPS 不可用
NOTIFYMSG NOCOMM    "UPS %s is unavailable"
# upsmon 父进程死亡，无法关闭
NOTIFYMSG NOPARENT  "upsmon parent process died - shutdown impossible"

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL+EXEC
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK   SYSLOG+EXEC
NOTIFYFLAG COMMBAD  SYSLOG+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+EXEC
NOTIFYFLAG NOCOMM   SYSLOG+EXEC
NOTIFYFLAG NOPARENT SYSLOG+EXEC

# SYSLOG+WALL+EXEC
# 记录系统日志+shutdown 通知+执行命令
```

### 通知配置

`/etc/nut/upssched.conf`

```conf
CMDSCRIPT /usr/bin/upssched-cmd
PIPEFN /var/lib/nut/upssched.pipe
LOCKFN /var/lib/nut/upssched.lock

# 等待 600 秒后通知断电
AT ONBATT * START-TIMER onbatt 600
AT ONLINE * CANCEL-TIMER onbatt online
AT LOWBATT * EXECUTE lowbatt
AT COMMBAD * START-TIMER commbad 30
AT COMMOK * CANCEL-TIMER commbad commok
AT NOCOMM * EXECUTE commbad
AT SHUTDOWN * EXECUTE powerdown
AT REPLBATT * EXECUTE replacebatt
AT NOPARENT * EXECUTE noparent
```

> 可延长 `ONBATT` 时间在对应脚本中执行 `upsmon -c fsd` 关机。一般情况下居民用电超过十分钟未恢复，停电时间就会很长。

### 通知脚本

`/usr/bin/upssched-cmd`

```bash
#!/bin/bash
case $1 in
    onbatt)
        logger -t upssched-cmd "UPS is on battery"
        /storage/scripts/SendMessage.py "UPS is on battery"
        /storage/scripts/other/kvm-manage stop
        ;;
    online)
        logger -t upssched-cmd "UPS is on battery"
        /storage/scripts/SendMessage.py "UPS is back on power"
        /storage/scripts/other/kvm-manage start
        ;;
    lowbatt)
        logger -t upssched-cmd "UPS battery is low, FSD"
        /storage/scripts/SendMessage.py "UPS battery is low, FSD"
        ;;
    commbad)
        /storage/scripts/SendMessage.py "Lost communication with UPS"
        ;;
    commok)
        /storage/scripts/SendMessage.py "Re-establish communication with UPS"
        ;;
    powerdown)
        /storage/scripts/SendMessage.py "UPS is shutting down the system"
        ;;
    replacebatt)
        /storage/scripts/SendMessage.py "UPS needs new battery"
        ;;
    noparent)
        /storage/scripts/SendMessage.py "upsmon parent process died"
        ;;
    *)
        logger -t upssched-cmd "Unrecognized command: $1"
        ;;
esac
```

这里是我的通知脚本，停电时会执行通知+关闭虚拟机。`logger -t upssched-cmd` 是写入系统日志的命令。

### 重启监控

```
systemctl restart nut-monitor.service
```

## 一些命令

列出 UPS 支持的命令 `upscmd -l ups`，这些命令 UPS 不一定完全支持。

```
# 关闭烦人的喇叭，再执行一次为打开
beeper.toggle - Toggle the UPS beeper
# UPS 关闭负载，有可能会造成 UPS 关机
load.off - Turn off the load immediately
# UPS 开启负载
load.on - Turn on the load immediately
# UPS 关机，来电后恢复
shutdown.return - Turn off the load and return when power is back
# UPS 关机，来电也不恢复开机
shutdown.stayoff - Turn off the load and remain off
# 停止 UPS 关机
shutdown.stop - Stop a shutdown in progress
# 测试电池
test.battery.start - Start a battery test
# 深度测试电池
test.battery.start.deep - Start a deep battery test
# 快速测试电池，切换一下电池供电
test.battery.start.quick - Start a quick battery test
# 停止测试电池
test.battery.stop - Stop the battery test
```

### 执行命令示例

```shell
upscmd -u admin -p masterpassword ups beeper.toggle
```

## 注意事项

1. 二手 UPS 如果是塑料壳的记得买带原包装的，别问我为什么。  
2. 自己做 RS232 线要跟着 DB9 插头上的线序标注接线，公母头的线序是镜像的。
3. 不要在市电断开时调试重启 `nut-server`，`nut-monitor` 检测不到 UPS 时会认为 UPS 关机了，导致迅速执行 FSD 关机。
4. 调试 UPS 时可调整监控配置中的关机命令，延长关机时间。
5. 调试时执行 `upsc ups` 命令，`ups.status` 字段如果存在 `FSD`，代表 UPS 可能会关机，如果执行 `shutdown.stop` 无效的情况下，可以试试重启 `nut-server`。

## 参考

1. [TrueNAS/FreeNAS使用山特MT系列UPS](https://www.izilzty.com/?post=7)
2. [Gentoo Wiki](https://wiki.gentoo.org/wiki/NUT)