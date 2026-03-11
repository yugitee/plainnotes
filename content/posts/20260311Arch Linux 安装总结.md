---
title: Arch Linux 安装总结
date: 2026-03-11T12:13:14+08:00
lastmod: 2026-03-11T12:13:17+08:00
keywords:
  - Arch Linux
  - 系统
description: 从零开始安装linux
tags:
categories:总结
draft: false
---


# Arch Linux 安装总结

本方案目标：

- 从零安装 Arch Linux
- 系统盘使用 **Btrfs**——方便回滚系统
- 数据盘使用 **ext4**
- 系统快照不会影响数据
- 使用 **swapfile**

# 一、磁盘规划

当前硬件：

```
nvme0n1 476G 系统盘  
sda 894G 数据盘  
```

最终结构：

```
nvme0n1 (系统盘)  
├─nvme0n1p1 1G EFI  
└─nvme0n1p2 剩余 Btrfs

sda (数据盘)  
└─sda1 全盘 ext4 -> /mnt/data

swap  
└─/swapfile
```

将数据盘分离：系统回滚不会影响笔记

# 二、Live 系统准备

## 制作启动盘

下载ISO镜像，使用ventoy制作镜像。（略）

## 启动

笔记本按F2进入bois，调整启动顺序，usb最前。保存+重启。

## 启动 LiveUSB 后：

同步时间：

```bash
timedatectl set-ntp true
````

确认网络：

```bash
ping archlinux.org #若是Wi-Fi，没网络需要iwctl进行联网
```

查看磁盘：

```bash
lsblk
```


# 三、磁盘分区

## 系统盘

```bash
cfdisk /dev/nvme0n1
```

创建：
> 选择磁盘、新建new--设置大小--设置type
> 注意需要用gpt格式。如果不是，推出重新进去，选择gpt

```
nvme0n1p1   1G     EFI System
nvme0n1p2   剩余   Linux filesystem
```

## 数据盘

```bash
cfdisk /dev/sda
```

创建：

```
sda1  全盘   Linux filesystem
```


# 四、创建文件系统

EFI：

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

系统盘：
>全部格式化为Btrfs

```bash
mkfs.btrfs -f /dev/nvme0n1p2
```

数据盘：

```bash
mkfs.ext4 /dev/sda1
```

---

# 五、创建 Btrfs 子卷结构

Btrfs 子卷类似于逻辑分区，但它们位于同一个文件系统内部。  
使用子卷的好处：

- 可以单独快照（snapshot）
- 可以单独挂载
- 方便系统回滚
- 结构更加灵活

常见的 Arch Btrfs 布局：

```

@  
@home  
@snapshots

````

## 5.1 临时挂载 Btrfs 分区

在创建子卷前，需要先挂载 Btrfs 分区：
>将mnt挂载到物理硬盘的第二分区——nvme0n1p2，第二分区就是上面根据分区表建立的。

```bash
mount /dev/nvme0n1p2 /mnt
````

说明：`/mnt` 是 Arch 安装过程的临时挂载点，之后系统会安装到 `/mnt` 中

## 5.2 创建 Btrfs 子卷

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
```

此时结构为：

```
/mnt
├── @
├── @home
└── @snapshots
```

这些子卷未来会分别挂载到：

```
/        -> @
/home    -> @home
/.snapshots -> @snapshots
```

## 5.3 卸载临时挂载

子卷创建完成后，卸载：

```bash
umount /mnt
```

原因：
目前的磁盘架构是：
Btrfs filesystem (top-level)
├── @
├── @home
└── @snapshots
如果在这里安装系统，那么系统就会存在与Btrfs的top-level中，那我们创建的子卷都没有用了。我们需要将系统安装在@中，那就**需要将top-level的挂载卸载掉，重新挂载子卷**。这是安装系统的需要。

# 六、重新挂载 Btrfs 系统分区

之前挂载的是 **整个 Btrfs 分区**，  
现在需要只挂载 **@ 子卷作为系统根目录**。

```bash
mount -o subvol=@,compress=zstd,noatime /dev/nvme0n1p2 /mnt
```

参数说明：

| 参数            | 作用         |
| ------------- | ---------- |
| subvol=@      | 使用 @ 子卷作为根 |
| compress=zstd | 启用压缩       |
| noatime       | 减少磁盘写入     |

这样 `/mnt` 就对应未来系统的 `/`。

## 6.1 创建系统目录结构

创建系统需要的目录：

```bash
mkdir -p /mnt/{boot,home,.snapshots,mnt/data}
```

目录用途：

| 目录          | 用途       |
| ----------- | -------- |
| /boot       | EFI 启动文件 |
| /home       | 用户目录     |
| /.snapshots | Btrfs 快照 |
| /mnt/data   | 数据盘挂载点   |

## 6.2 挂载其他子卷

将子卷挂载到对应目录：

```bash
mount -o subvol=@home,compress=zstd,noatime /dev/nvme0n1p2 /mnt/home
```

```bash
mount -o subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
```

**访问 /mnt/home= 访问 nvme0n1p2 中的 @home 子卷**
**其实 /mnt = /** ——而这个mnt其实也就是一个临时的目录，它可以是任何目录名称。只不过arch wiki中使用mnt，然后都用mnt也好做教程。

现在 Btrfs 的结构已经完成。也完成了挂载根分区的步骤。

# 七、挂载 EFI 分区

EFI 分区用于存放 **系统启动文件**。

```bash
mount /dev/nvme0n1p1 /mnt/boot
```

原因：

安装 Bootloader（例如 GRUB）时  
必须将 EFI 分区挂载到 `/boot`。

EFI 文件系统必须是 **FAT32**。（前面1G的空间，已经格式化为FAT32）


# 八、挂载数据盘

将第二块硬盘作为 **独立数据盘**，这样做的好处是数据和系统分离，系统出问题数据也没事。所有的笔记、照片、视频都可以安全存放。

```bash
mount /dev/sda1 /mnt/mnt/data
```


至此，**目前文件系统的架构是**：

```
/mnt
├── boot        -> mount /dev/nvme0n1p1
├── home        -> mount Btrfs subvol @home
├── .snapshots  -> mount Btrfs subvol @snapshots
└── mnt
    └── data    -> mount /dev/sda1
```


# 九、安装 Arch 基础系统

现在已经完成磁盘准备，可以开始安装系统。

```bash
pacstrap /mnt base linux linux-firmware btrfs-progs sudo nano networkmanager grub efibootmgr
```

命令解释：

|包|作用|
|---|---|
|base|Arch基础工具|
|linux|Linux内核|
|linux-firmware|硬件驱动|
|btrfs-progs|Btrfs工具|
|sudo|权限管理|
|nano|文本编辑器|
|networkmanager|网络管理|
|grub|启动管理|
|efibootmgr|EFI管理|

`pacstrap` 的作用是：**把 Arch 系统安装到 /mnt 目录中**。

# 十、生成 fstab 文件

fstab 文件用于定义 **系统启动时自动挂载的磁盘**。

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

参数说明：

| 参数  | 含义      |
| --- | ------- |
| -U  | 使用 UUID |
``  > > | 追加到文件 |`

将挂载的信息追加写到fatab中，其中，文件系统用UUID来表示，因为这个一直不会变。

## 10.1 检查 fstab

```bash
cat /mnt/etc/fstab
```

应该包含类似内容：

```
UUID=xxx / btrfs subvol=@
UUID=xxx /home btrfs subvol=@home
UUID=xxx /mnt/data ext4
```

如果没有 `/mnt/data`  
说明数据盘没有正确挂载。

# 十一、进入新系统

现在 `/mnt` 中已经是完整的 Linux 系统。此时[[安装过程中，什么时候拔U盘？|为什么不能拔除U盘？？]]
执行：

```bash
arch-chroot /mnt
```

`chroot` 的作用是：将当前环境切换到`/mnt`

之后所有操作都在新系统中进行。

# 十二、系统基础配置

## 设置时区

执行：

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

然后同步硬件时间：

```bash
hwclock --systohc
```

## 设置语言

编辑：

```bash
nano /etc/locale.gen
```

取消注释：

```
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

生成语言：

```bash
locale-gen
```

设置默认语言：

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

# 十三、设置主机名

执行：

```bash
echo archlinux > /etc/hostname
```

## 配置 hosts

编辑：

```bash
nano /etc/hosts
```

写入：

```
127.0.0.1 localhost
::1 localhost
127.0.1.1 archlinux.localdomain archlinux
```

# 十四、设置 root 密码

执行：

```bash
passwd
```

输入 root 密码。

# 十五、安装 GRUB 启动器

安装 GRUB 到 EFI：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

生成启动配置：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

作用：**让电脑开机能够启动 Linux。**

# 十六、启用网络服务

Arch 默认不自动联网。启用 NetworkManager：

```bash
systemctl enable NetworkManager
```

系统启动后会自动连接网络。

# 十七、创建普通用户

Linux 日常使用 **不应该使用 root**。

创建用户：

```bash
useradd -m -G wheel we
```

设置密码：

```bash
passwd we
```

## 启用 sudo

执行：

```bash
EDITOR=nano visudo
```

找到：

```
%wheel ALL=(ALL:ALL) ALL
```

取消前面的 `#`。

这样 `we` 用户就可以使用 `sudo`。

至此：系统安装已经基本完成。  
接下来：

```
exit
umount -R /mnt
reboot
```

此时拔掉 USB，系统就可以启动。

# 十八、系统启动后

## 检查

查看磁盘：

```bash
lsblk
```

检查 swap：

```bash
swapon --show
```

检查挂载：

```
/boot
/
/home
/mnt/data
```


## swapfile  

文件位置：放在系统根目录 `/` 下（也就是 Btrfs 子卷 `@` 中）

```bash
btrfs filesystem mkswapfile --size 8g /swapfile
```

启用：

```bash
swapon /swapfile
```

加入 fstab：

```bash
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
```

这样 swap 就随系统启动自动启用，不占用额外分区，同时 Btrfs 的子卷快照也不会包括 swapfile。

# 结语

[arch wiki](https://wiki.archlinuxcn.org)中已经讲明白了所有的安装流程，我之所以做这么一个总结，主要是让自己再次梳理挂载点、安装、硬盘等关系，这对理解系统有帮助。
这篇文章向大家提供了一次实践的过程展示，让大家了解一下具体过程，虽然有点多。其实arch的安装与其他相比就是少了一个图形界面，底层的东西都是一样的。