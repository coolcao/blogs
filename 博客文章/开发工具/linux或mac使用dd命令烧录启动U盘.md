---
title: linux或mac使用dd命令烧录启动U盘
date: 2016-11-10 22:01:46
tags: [linux]
categories:
- 闲聊杂谈
- 技多不压身
---

linux下可以使用dd命令简单烧录iso镜像到U盘，做可启动安装U盘。

<!-- more -->

## 下载iso镜像
下载iso文件(live启动盘，普通的安装盘可能不行),并将其放到目录下，例如：
```
/Users/coolcao/Downloads/debian-live-8.4.0-amd64-gnome-desktop.iso
```
## 查看USB设备编号
### mac
```
diskutil list
```
mac下可以看到类似下面的数据：
```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:          Apple_CoreStorage Macintosh HD            250.1 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3

/dev/disk1 (internal, virtual):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:                            Macintosh HD           +249.8 GB   disk1
                                 Logical Volume on disk0s2
                                 38C1EAE0-D413-4203-A64F-DEE8B6FA99C1
                                 Unencrypted

/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *4.0 GB     disk2
   1:                       0xEF                         31.5 MB    disk2s2
```
### linux
```
fdisk -l
```
linux下看到类似如下的数据：
```
Disk /dev/sdb: 3.7 GiB, 4003463168 bytes, 7819264 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x2609d887

设备       启动 Start    末尾    扇区  Size Id 类型
/dev/sdb1  *        0 1734655 1734656  847M  0 空
/dev/sdb2         212   61651   61440   30M ef EFI (FAT-12/16/32)
```
根据U盘的容量，很容易判断出U盘的盘符。
这里我使用mac系统。我的设备是`/dev/disk2`

## dd命令烧录
```
sudo dd if=/Users/coolcao/Downloads/debian-live-8.4.0-amd64-gnome-desktop.iso of=/dev/disk2 bs=1m
```




> mac下如果遇到 `dd: /dev/disk2: Resource busy`错误，先使用`diskutil unmountDisk /dev/disk2`命令卸载磁盘再烧录。
