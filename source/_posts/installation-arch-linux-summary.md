---
title: 安装Arch Linux小结
date: 2021-07-28 13:48:38
tags:
---

<img src="https://archlinux.org/static/logos/archlinux-logo-dark-1200dpi.b42bd35d5916.png" style="zoom:5%;" />

本人跟随[Arch Linux官方安装指南](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))操作过程的一些记录。

## 准备

>  下载ISO文件并启动到 Live 环境，注意[安全启动（Secure Boot）](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)/Secure_Boot_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))的问题。

[官方下载地址](https://archlinux.org/download/)

### 验证启动模式(非必须)

请用下列命令列出 [efivars](https://wiki.archlinux.org/index.php/Efivars) 目录：

```shell
ls /sys/firmware/efi/efivars
```

目录存在，操作系统使用UEFI模式启动，反之使用传统模式启动(BIOS,CSM)。

### 网络连接

**安装过程需要连接互联网**，Live环境默认开启DHCP，其它方式可以参考[连接到因特网](https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%BF%9E%E6%8E%A5%E5%88%B0%E5%9B%A0%E7%89%B9%E7%BD%91)，

### 更新系统时间

```shell
timedatectl set-ntp true
```

### 建立硬盘分区

> 使用`fdisk`分区，可以查看这篇[博文](https://www.cnblogs.com/lsgxeva/p/9610003.html)有详细的使用说明

查看分区列表

```shell
fdisk -l
```

![](https://pic.liaoheng.me/hc-pic/2024/02/93503db3f240a11f738fc5c37e0287a2.png)

其中`/dev/sda`代表第一块物理磁盘(`/dev/sdb,/dev/sdc`与之类推)

操作对应磁盘(`/dev/sda`)分区

```shell
fdisk /dev/sda
# 进入fdisk应用命令行：
g #如果没有分区表或者是MBR表先删除，创建一个新的空GPT分区表
n #创建新的分区
t #改变分区类型
w #保存并退出
```

![](https://pic.liaoheng.me/hc-pic/2024/02/4811eca3d0b3a9f324247b14d39e571b.png)

新建分区之后需要指定分区类型，使用命令`t {partition number }`，可以参考下表配置分区：

| Mount point | Partition | [Partition type GUID](https://en.wikipedia.org/wiki/GUID_Partition_Table#Partition_type_GUIDs) |
| ----------- | --------- | ------------------------------------------------------------ |
| /boot       | /dev/sda1 | `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`: [EFI system partition](https://wiki.archlinux.org/index.php/EFI_system_partition) |
| [SWAP]      | /dev/sda2 | `0657FD6D-A4AB-43C4-84E5-0933C84B4F4F`: Linux [swap](https://wiki.archlinux.org/index.php/Swap) |
| /home       | /dev/sda3 | `933AC7E1-2EB4-4F13-B844-0E14E2AEF915`: Linux /home          |
| /           | /dev/sda4 | `4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709`: Linux x86-64 root (/) |

通过`t`命令可以查看更多类型，PgUp向上,PgDn向下,q退出。

### 格式化分区

```shell
mkfs.fat -F32 /dev/sda1 #EFI system partition(ESP)
mkswap /dev/sda2 #swap
mkfs.ext4 /dev/sda3 #/home
mkfs.ext4 /dev/sda4 #/
```

### 挂载分区

将根磁盘卷 [挂载](https://wiki.archlinux.org/index.php/Mount) 到 `/mnt`，例如：

```shell
mount /dev/sda4 /mnt #挂载/
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot #挂载ESP
mkdir /mnt/home
mount /dev/sda3 /mnt/home #挂载/home
swapon /dev/sda2 #挂载swap
```

## 安装系统

### 选择一个合适的镜像(可选)

[非官方镜像列表](https://wiki.archlinux.org/title/Mirrors_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E4%B8%AD%E5%9B%BD)

```shell
nano /etc/pacman.d/mirrorlist
---
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch #文件第一排写入
```

### 安装配置

这几个包按照需求选择是否添加 [nano](https://wiki.archlinux.org/title/Nano_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)), [grub](https://wiki.archlinux.org/title/GRUB_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)), [efibootmgr](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#efibootmgr), [iwd](https://wiki.archlinux.org/title/Iwd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

```shell
pacstrap /mnt base linux linux-firmware nano grub efibootmgr iwd
genfstab -U /mnt >> /mnt/etc/fstab  #定义磁盘分区
```

### 通过Chroot进入安装系统

```sh
arch-chroot /mnt # 进入安装好的arch linux系统
```
在当前Live Linux中通过[Chroot](https://wiki.archlinux.org/title/Chroot_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))进入我们挂在到/mnt下面的Arch Linux系统中(也就是我们这次安装的系统)，以下操作均在其中

### 时间/时区

```sh
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  #设置时区
hwclock --systohc #硬件时间设置为UTC
```

### 本地化

编辑`/etc/locale.gen` 选择 [地区](https://wiki.archlinux.org/index.php/Locale_(简体中文)) ：

```sh
nano /etc/locale.gen #移除需要的地区前的#号
---
locale-gen #更新配置
```

然后创建`/etc/locale.conf` 文件，并添加 [ LANG 变量](https://wiki.archlinux.org/index.php/Locale#Setting_the_system_locale)：

```sh
nano /etc/locale.conf
---
LANG=en_GB.UTF-8 #推荐使用，而不是zh_CN
```

### hostname&hosts

```sh
#创建/etc/hostname文件
nano /etc/hostname
---
LHCOMPUTER #写入主机名称

#编辑/etc/hosts
nano /etc/hosts
---
127.0.0.1    localhost
::1        localhost
127.0.1.1    LHCOMPUTER.localdomain    LHCOMPUTER #添加
```

### 设置Root用户密码

```sh
passwd
```

### 安装引导程序

前面的步骤已经基本完成对启动系统的配置，现在需要计算机引导启动系统，我选择使用[GRUB](https://wiki.archlinux.org/index.php/GRUB_(简体中文))，以下是在EFI 系统分区下的命令：

```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### 完成

输入 `exit` 或按 `Ctrl+d` 退出 Chroot 环境。
可选用 `umount -R /mnt` 手动卸载被挂载的分区。
最后执行 `reboot` 重启系统。

## 使用新系统前的配置

### 选择网络管理程序

系统默认已安装[systemd-networkd](https://wiki.archlinux.org/index.php/Systemd-networkd_(简体中文))，以下步骤配置并开启。

```shell
$ networkctl #获取有关网络链接的状态信息
IDX LINK             TYPE               OPERATIONAL SETUP     
1 lo               loopback           carrier     unmanaged 
2 ens33            ether              routable    unmanaged 
3 wlp2s0           wlan               off         unmanaged 
4 vmnet1           ether              routable    unmanaged 
5 vmnet8           ether              routable    unmanaged 
5 links listed.
```

其中ens33为有线网卡，wlp2s0为无线网卡。

#### 配置有线

```sh
nano /etc/systemd/network/dhcp.network
---
[Match]
Name=ens33
 
[Network]
DHCP=true
#Address=192.168.1.20/24
#Gateway=192.168.1.1
#DNS=119.29.29.29
#DNS=8.8.8.8
```

#### 配置无线

需要先安装[iwd](https://wiki.archlinux.org/title/Iwd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))包，使用命令连接对应无线网络。

1. 要进入交互式提示符（interactive prompt），执行：`$ iwctl`
2. `[iwd]# station device scan`
3. `[iwd]# station device get-networks`
4. `[iwd]# station device connect SSID`，会提示输入密码

完成后，配置网络

```sh
nano /etc/systemd/network/wireless.network
---
[Match]
Name=wlp2s0
 
[Network]
DHCP=true
```
#### 开启自动服务

```sh
systemctl enable --now systemd-networkd.service
systemctl enable --now systemd-resolved.service #DNS
```

### 添加新用户

```sh
groupadd liaoheng
useradd -m -g liaoheng liaoheng
passwd liaoheng
```

### 安装`sudo`

```sh
pacman -S sudo
nano /etc/sudoers#下面内容到文件末尾
---
Defaults targetpw
liaoheng ALL=(ALL) ALL
```

### 开机时打开 `Num Lock`

```sh
sudo pacman -S systemd-numlockontty
sudo systemctl enable --now numLockOnTty.service
```

### 软件仓库(可选)

> 可以登录`liaoheng`账户

```sh
sudo pacman -Syu
```

添加Arch Linux Chinese Community Repository，编辑`/etc/pacman.conf`，添加

```sh
sudo nano /etc/pacman.conf
---
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```

导入PGP keys:

> [失败解决办法](https://github.com/archlinuxcn/repo/issues/522)

```sh
sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring
```

### 图形界面

#### [GNOME](https://wiki.archlinux.org/index.php/GNOME_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

```sh
sudo pacman -S gnome gnome-extra gnome-tweak-tool
sudo pacman -S networkmanager
sudo systemctl enable NetworkManager
sudo systemctl enable --now gdm.service
```

## 参考

- [Installation guide](https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [Partitioning](https://wiki.archlinux.org/index.php/Partitioning_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%B8%83%E5%B1%80%E7%A4%BA%E4%BE%8B)
- [EFI_system_partition](https://wiki.archlinux.org/index.php/EFI_system_partition)
- [fstab ](https://wiki.archlinux.org/index.php/Fstab_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [GRUB](https://wiki.archlinux.org/index.php/GRUB_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))