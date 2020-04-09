---
layout: post
title: "为树莓派 4 安装纯净的 arm64 Debian GNU/Linux 系统"
date: 2020-04-09
author: Cyanoxygen
cover: '/assets/img/hero.png'
tags: Debian Raspberry Pi
---

# 为树莓派 4 安装 arm64 的 Debian

[refer]:https://hackaday.com/2020/01/28/raspberry-pi-4-benchmarks-32-vs-64-bits/
[raspbian-download]:https://www.raspberrypi.org/downloads/raspbian/
[forumlink_sound]:https://www.raspberrypi.org/forums/viewtopic.php?t=263942

最近入手了一块树莓派 4B 4GB 版，打算将其作为 NAS 和数据库服务器用，~~放着也是闲着~~。
由于它使用的 SoC 是 arm64 架构，所以一直想为其安装一个纯 64 位系统。在看过网上的一篇[评测][refer]之后，为其安装 64 位系统的念头更加强烈。

我是一个 Debian Puriest, 因为 ~~Raspbian 并不是纯正的 Debian~~ Raspbian 官方并没有提供 64 位的 Userland，但是提供了 64 位内核，所以我决定使用 `debootstrap` 工具现场将 Raspbian 替换为 Debian aarch64。

> 本文参考了：https://david.wragg.org/blog/2020/01/installing-64-bit-debian-on-rpi.html

## 前期准备

为了完成 `debootstrap` 过程，你至少需要：

- Raspbian 镜像（任何版本都可以！）
- 另一台 Linux 电脑（或者虚拟机，推荐使用 Debian）
  - 如果你没有电脑，你可以把另一张安装了系统的 SD 卡插入树莓派，之后挂载这张 SD 卡。**请注意，请不要将分区混淆，以免发生意外情况。**
- 网络连接。
- `debootstrap` 程序，你可以在大多数发行版仓库中找到它。
- 树莓派需要外接显示器和键盘操作（你或许可以将控制台设置为串口）。
  
## 注意事项

- 尽管是使用 Raspbian 内核，3D 和硬件加速也许不可用（因为我并没有试过安装桌面）。
- 请在运行 `debootstrap` 前做好相应的设置。
- 此操作会升级你的固件。
- 安装过程中无法使用 SSH。**你需要外接显示器和键盘，或强制仅串口 Console**。
  - 如果你没有显示器，可以盲打，或者连接你的串口，请在准备安装时按照步骤来。第二次重启后的步骤只需要小心敲打命令即可通过。
- `debootstrap` 是一个可以在某个文件系统中部署最小 Debian 系统的工具，是 Debian 安装器的一部分。它可以运行在当前运行的 Linux 上，支持跨架构安装。它分为两个步骤，第一个步骤负责下载软件包并准备安装环境；第二阶段负责安装基本系统和软件包。


## 大致步骤
1. 刷写 Raspbian
2. 更新内核和固件
3. 启用 64 位内核
4. 备份重要文件（模块和固件等）
5. 擦除分区
6. 运行 `debootstrap`
7. 完成安装和设置
 
## 让我们开始吧！

### 0x00 Raspbian 安装和配置

首先，[前往下载 Raspbian][raspbian-download]。使用你喜欢的工具将其写入 SD 卡并安装在树莓派上。

将网线连接至树莓派。如果没有，则可以在配置之后手动连接 Wi-Fi。

待树莓派启动后，立即运行配置工具。

```shell
sudo raspi-config
```
你一定需要做的事情就是设置 Wi-Fi 区域。Wi-Fi 区域如果不被设置，在安装完系统之后很可能无法工作。你也可以进行你想做的其他配置，比如配置声音输出等。**如果你没有连接以太网，请使用 `raspi-config` 配置好你的 Wi-Fi 连接！**

配置完毕，接下来更换镜像源（如果需要），然后升级系统：

```shell
sudo apt update && sudo apt upgrade
```

现在你需要将树莓派的内核和固件升级到最新测试版：

```shell
sudo rpi-update
```

```
 *** Raspberry Pi firmware updater by Hexxeh, enhanced by AndrewS and Dom
 *** We're running for the first time
 *** Backing up files (this will take a few minutes)
 *** Backing up firmware
 *** Backing up modules 4.19.97-v7l+
#############################################################
WARNING: 'rpi-update' updates to pre-releases of the linux 
kernel tree and Videocore firmware.

'rpi-update' should only be used if there is a specific 
reason to do so - for example, a request by a Raspberry Pi 
engineer.

DO NOT use 'rpi-update' as part of a regular update process.

##############################################################
Would you like to proceed? (y/N)
```
在这里输入 `y` 并回车，开始更新。更新可能需要等待较长时间。
在内核升级完之后，你需要以 root 身份编辑 `/boot/config.txt` 以启用 64 位内核：

```shell
sudo -s
echo arm_64bit=1 >> /boot/config.txt
exit
```

在更换新内核之前可以查看一下 `uname` 的输出：

```
pi@raspberrypi:~ $ uname -a
Linux raspberrypi 4.19.97-v7l+ #1294 SMP Thu Jan 30 13:21:14 GMT 2020 armv7l GNU/Linux
```

`cat /proc/cpuinfo` :

```
processor	: 0
model name	: ARMv7 Processor rev 3 (v7l)
BogoMIPS	: 108.00
Features	: half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm crc32 
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3
```

`time echo 'scale=2000;4*a(1)' | bc -l` 计算圆周率（仅供娱乐）:

```
real	0m3.827s
user	0m3.798s
sys	    0m0.014s
```

现在关闭你的树莓派，拔下 SD 卡，将其插入读卡器，连接到另一台 Linux 机器。

---

### 0x01 备份重要文件

在正式安装之前，我们需要备份一些重要的配置文件。

Debian 原生不提供（？）适用于树莓派 4 的内核模块和固件，所以我们只能使用现成的。
- `fstab` 文件也需要备份，因为 `debootstrap` 并不会自动生成这些文件。
- `/etc/wpa_supplicant/wpa_supplicant.conf` 也需要备份，因为里面包含有 Wi-Fi 配置信息。
  
找一个临时文件夹，然后进入 `root` Shell。假设你的 `rootfs` 设备为 `/dev/sda2` ，挂载在 `/mnt/sdroot` :

```shell
sudo -s     # 进入 root Shell
export TMPDIR=/tmp  # 设置临时文件夹，当然你可以手动新建一个文件夹
mkdir /mnt/sdroot   # 创建目录
mount /dev/sda2 /mnt/sdroot     # 挂载根文件系统

# 备份内核模块和固件
cp -a /mnt/sdroot/lib/firmware /mnt/sdroot/lib/modules $TMPDIR/
# 备份 fstab 和 wpa_supplicant.conf
cp -a /mnt/sdroot/etc/fstab $TMPDIR/fstab
cp -a /mnt/sdroot/etc/wpa_supplicant/wpa_supplicant.conf $TMPDIR/wpa_supplicant.conf

# 卸载文件系统
umount /mnt/sdroot
exit
```

- 如果你需要在安装完成之后立即获得网络连接，你可能需要提前写一个。

-----

### 0x02 安装 Debian

备份完成，我们可以抹除之前的数据了：

```shell
sudo mkfs.ext4 -L debian /dev/sda2
```

现在请挂载你的文件系统，执行 `debootstrap` 来安装一个最小的 Debian 系统：

```shell
sudo mount /dev/sda2 /mnt/sdroot
sudo debootstrap --include=wpasupplicant,dbus --arch=arm64 --foreign buster /mnt/sdroot
```

如果你需要镜像源来获得更快的下载速度，请执行：

```shell
sudo debootstrap --include=wpasupplicant,dbus --arch=arm64 --foreign buster /mnt/sdroot http://mirrors.ustc.edu.cn/debian/
```

如果中途失败，你可以重新执行本条命令。

等待安装完成，然后来做一些配置。首先我们需要挂载启动分区（即那个 300M 大小的 FAT32 分区）。

```shell
sudo mkdir /mnt/boot
sudo mount /dev/sda1 /mnt/boot
```

之后我们需要恢复之前备份的文件，假设备份的文件在 `/tmp` ：

```shell
sudo -s
export TMPDIR=/tmp
# 恢复备份的模块和固件
cp -a $TMPDIR/firmware /mnt/sdroot/lib/firmware
cp -a $TMPDIR/modules /mnt/sdroot/lib/modules
# 恢复 fstab 和 wpa_supplicant.conf
cp -a $TMPDIR/fstab /mnt/sdroot/etc/fstab
cp -a $TMPDIR/wpa_supplicant.conf /mnt/sdroot/etc/wpa_supplicant
```

这时候请不要着急，我们还有一些工作要做。要在本地网络上区别你的树莓派，你需要重新指定主机名：

```shell
echo yourhostname > /mnt/sdroot/etc/hostname
```

接下来你需要检查 `fstab` 文件以及 `/mnt/boot/cmdline.txt` 中的 `root` 参数，根文件系统的标识点必须与 `blkid` 中相应分区的的输出结果 `PARTUUID` 一致。否则你的树莓派将无法进入系统。

- 如果你使用串口，请编辑 `/mnt/boot/cmdline.txt` ，将 `console=tty1` 改为 `console=ttyS0` 。之后你可以以 `115200, 8N1` 的参数连接串口。

在上述步骤全部完成之后，卸载文件系统，将安装了 Debian 的 SD 卡插入树莓派启动。

```shell
sudo umount /mnt/sdroot /mnt/boot ; sync ; sync
```

---

### 0x03 第二阶段

如果你连接了显示器或串口，你应看到输出的内核消息。等屏幕不再滚动，敲一下回车，一个非常简陋的 Shell 就会被激活。这个 Shell 的简陋程度是：它不支持命令历史记录，不支持 Tab 补全，你仅仅能按下的其他按键只有退格键和回车。所以每一条命令都要小心执行。

现在我们需要挂载一些必要的文件系统，然后进行 `debootstrap` 的第二阶段。第二阶段会解包基本系统，安装核心组件。

```shell
mount /proc
mount -o remount,rw /dev/root /
/debootstrap/debootstrao --second-stage
```

此阶段同样需要等待一段时间。如果你没有连接显示器或串口，你需要等待至少 5 分钟或直到 ACT LED 不再亮起或闪动。
接下来的步骤非常重要，我们需要设置 root 用户的密码，否则重启后你无法登录。

```shell
passwd
```

输入你的 `root` 密码，敲回车，再输入一遍密码敲回车。密码设置完成后，卸载文件系统，同步写入并重启：

```shell
mount -r -o remount -f /dev/root ; sync ; sync ; reboot -f
```

---

### 0x04 配置 Debian

到此时我们的纯 Debian 就安装好了，尽管它还是个最小安装，我们依然可以通过软件包来增加功能来满足基本需求。

首先配置网络（如果你之前备份了 `interfaces` 文件并恢复了它，可以跳过）：

```sh
nano /etc/network/interfaces
```
对于一个连接了有线网络，并且网络内有 DHCP 的系统，你可以这样填写：

```
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp
```
对于一个配置了有线网络的系统：
```
auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
	wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

保存文件，执行 `ifup` 来使设置生效：

```
ifup eth0
```

过几秒，你的树莓派应该连接到了网络：

```
ip addr
```

连接到了网络，你现在可以直接像以前一样操作 Debian 了：

```
apt update && apt upgrade
```
配置本地化、时区和键盘布局：

```
dpkg-reconfigure tzdata
apt install locales
dpkg-reconfigure locales
apt install console-setup
dpkg-reocnfigure keyboard-configuration
```

然后安装基本系统工具：

```
apt update

tasksel install standard ssh-server

apt install accountsservice bash-completion ca-certificates curl git htop perl sudo dnsutils net-tools openssl neofetch cowsay bc
```
添加用户：

```
adduser user
usermod -aG sudo user  # 将其添加至 sudo 组中
```

重启你的树莓派，恭喜你，大功告成，你的 Debian 基本系统已经安装好了。
接下来看看它的信息：

```
$ neofetch

       _,met$$$$$gg.          user@hostname 
    ,g$$$$$$$$$$$$$$$P.       ------------------ 
  ,g$$P"     """Y$$.".        OS: Debian GNU/Linux 10 (buster) aarch64 
 ,$$P'              `$$$.     Host: Raspberry Pi 4 Model B Rev 1.2 
',$$P       ,ggs.     `$$b:   Kernel: 4.19.113-v8+ 
`d$$'     ,$P"'   .    $$$    Uptime: 17 mins 
 $$P      d$'     ,    $$P    Packages: 721 (dpkg) 
 $$:      $$.   -    ,d$$'    Shell: bash 5.0.3 
 $$;      Y$b._   _,d$P'      Terminal: /dev/pts/0 
 Y$$.    `.`"Y$$$$P"'         CPU: (4) @ 1.500GHz 
 `$$b      "-.__              Memory: 193MiB / 3755MiB 
  `Y$$
   `Y$$.                                              
     `$$b.
       `Y$$b.
          `"Y$b._
              `"""
```

`uname -a` :

```
Linux hostname 4.19.113-v8+ #1300 SMP PREEMPT Thu Mar 26 17:04:40 GMT 2020 aarch64 GNU/Linux
```

`cat /proc/cpuinfo` :

```
processor	: 0
BogoMIPS	: 108.00
Features	: fp asimd evtstrm crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3
```

~~再次娱乐一下，计算圆周率~~ `time echo 'scale=2000;4*a(1)' | bc -l` :

```
real	0m11.069s
user	0m11.053s
sys 	0m0.014s
```

## 后日谈

~~我不会告诉你这个词从哪里学到的~~ 不同的人有不同的需求，我可以把我遇到的问题一一列出：

### Q: 我想输出声音，怎么办？

A：你可以安装声音支持包。但是仍然需要一些配置。

```
sudo apt install alsa-base pulseaudio alsa-utils mpg123
```
然后确认 `/boot/config.txt` 中有该项内容并且并没有被注释掉：

```
dtparam=audio=on
```
重启树莓派，执行 `aplay -lL` 看看是否有 3 个输出设备。

接下来执行 `pactl list sinks`，如果你得到了一个单声道（Mono）输出，请按照以下步骤执行：
  - 编辑 `/usr/share/pulseaudio/alsa-mixer/profile-sets/default.conf`，将 `[Mapping analog-mono]` 一节全部使用分号 `;` 注释掉：

    ```
       104	;[Mapping analog-mono]
       105	;device-strings = hw:%f
       106	;channel-map = mono
       107	;paths-output = analog-output analog-output-lineout analog-output-speaker analog-output-headphones analog-output-headphones-2 analog-output-mono
       108	;paths-input = analog-input-front-mic analog-input-rear-mic analog-input-internal-mic analog-input-dock-mic analog-input analog-input-mic analog-input-linein analog-input-aux analog-input-video analog-input-tvtuner analog-input-fm analog-input-mic-line analog-input-headset-mic
       109	;priority = 7
       110	
    ```
    - 重启你的设备，再次执行 `pactl list sinks` 你应该会得到双声道输出。

### Q：如何切换输出设备？

通过执行 `amixer cset numid=3 1` 可以将输出切换到 3.5mm 端口。
类似的，将值 `1` 改为 `2` 即可切换到 HDMI1，`3` 则为 HDMI2。

### Q：我的输出没有声音 / 太小了！

A：你的输出设备可能被静音或者输出级别太低。
- 运行 `alsamixer` 。
- 按下 <kbd>F6</kbd>，选择声卡，回车。检查有没有设备被静音（竖条下面显示 `MM` 即被静音）。
- 如果有，按 <kbd>←</kbd> <kbd>→</kbd> 键将高亮移动到那个输出，按下 <kbd>M</kbd> 。
- 如果你看到了白色条很低的设备，移动到它上面，按 <kbd>↑</kbd> <kbd>↓</kbd> 调整音量。
待调整到合适的音量后，按 <kbd>ESC</kbd> 退出。

Q：我想把声音强制输出到 3.5mm 口，如何操作？

A：确实可以做到。对内核模块 `snd_bcm2835` 指定参数即可让声卡只有一个输出设备。

- 按照[论坛页面][forumlink_sound]的指引，编辑 `/boot/cmdline.txt` （注意，该文件只能有一行），在文件尾部追加以下内容：
  ```
  snd_bcm2835.enable_headphones=1 snd_bcm2835.enable_hdmi=1 snd_bcm2835.enable_compat_alsa=0
  ```
- 重启树莓派，执行 `aplay -lL`，你会发现输出设备变成了一个。

### Q：树莓派 4B 的千兆网卡是真的么？

A：的确是真的！你真的可以跑出至少 990Mbps 的下载速度。
- 以下是台式机和树莓派之间 `iperf3` 的结果。

    ```
    $ iperf3 -c 192.168.1.4 -p 2333
    Connecting to host 192.168.1.4, port 2333
    [  5] local 192.168.1.254 port 49212 connected to 192.168.1.4 port 2333
    [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
    [  5]   0.00-1.00   sec   113 MBytes   949 Mbits/sec    0    246 KBytes       
    [  5]   1.00-2.00   sec   112 MBytes   942 Mbits/sec    0    246 KBytes       
    [  5]   2.00-3.00   sec   112 MBytes   942 Mbits/sec    0    280 KBytes       
    [  5]   3.00-4.00   sec   112 MBytes   942 Mbits/sec    0    291 KBytes       
    [  5]   4.00-5.00   sec   112 MBytes   944 Mbits/sec    0    291 KBytes       
    [  5]   5.00-6.00   sec   112 MBytes   944 Mbits/sec    0    310 KBytes       
    [  5]   6.00-7.00   sec   112 MBytes   938 Mbits/sec    0    310 KBytes       
    [  5]   7.00-8.00   sec   113 MBytes   946 Mbits/sec    0    310 KBytes       
    [  5]   8.00-9.00   sec   112 MBytes   939 Mbits/sec    0    310 KBytes       
    [  5]   9.00-10.00  sec   112 MBytes   943 Mbits/sec    0    310 KBytes       
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bitrate         Retr
    [  5]   0.00-10.00  sec  1.10 GBytes   943 Mbits/sec    0             sender
    [  5]   0.00-10.04  sec  1.10 GBytes   938 Mbits/sec                  receiver

    iperf Done.
    ```

- 以下是树莓派与台式机之间 `iperf3` 的结果。

    ```
    $ iperf3 -c 192.168.1.254 -p 2333
    Connecting to host 192.168.1.254, port 2333
    [  5] local 192.168.1.4 port 54788 connected to 192.168.1.254 port 2333
    [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
    [  5]   0.00-1.00   sec  70.4 MBytes   590 Mbits/sec    0    373 KBytes       
    [  5]   1.00-2.00   sec  67.7 MBytes   568 Mbits/sec    0    373 KBytes       
    [  5]   2.00-3.00   sec  68.6 MBytes   575 Mbits/sec    0    373 KBytes       
    [  5]   3.00-4.00   sec  68.6 MBytes   575 Mbits/sec    0    373 KBytes       
    [  5]   4.00-5.00   sec  68.2 MBytes   572 Mbits/sec    0    373 KBytes       
    [  5]   5.00-6.00   sec  69.3 MBytes   582 Mbits/sec    0    373 KBytes       
    [  5]   6.00-7.00   sec  69.5 MBytes   583 Mbits/sec    0    373 KBytes       
    [  5]   7.00-8.00   sec  68.4 MBytes   573 Mbits/sec    0    373 KBytes       
    [  5]   8.00-9.00   sec  68.7 MBytes   576 Mbits/sec    0    373 KBytes       
    [  5]   9.00-10.00  sec  68.2 MBytes   573 Mbits/sec    0    373 KBytes       
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bitrate         Retr
    [  5]   0.00-10.00  sec   688 MBytes   577 Mbits/sec    0             sender
    [  5]   0.00-10.04  sec   686 MBytes   573 Mbits/sec                  receiver

    iperf Done.
    ```


在文章的最后，希望大家能多多支持原文章作者和我！Enjoy your Pure Debian experience with your Raspberry Pi!

Copyright (C) Cyanoxygen, published under CC BY-SA NC 3.0.
