---
date: '2024-03-12T18:04:24+08:00'
title: '硬盘挂载'
summary: "Linux挂载硬盘操作流程"
tags: ["Linux"]
---

### 1.查看硬盘挂载情况
``` bash
fdisk -l
```

### 2.查看当前分区情况
``` bash
df -l
```

### 3.给新硬盘添加新分区（可选）
``` bash
_fdisk _/dev/vdb
```

### 4.分区完成，查询所有设备的文件系统类型
``` bash
blkid
```

### 5.格式化分区
先查看当前系统支持格式化成什么类型，输入mkfs，然后按两下tab键
``` bash
mkfs.xfs /dev/vdb1
blkid
```

### 6.挂载
``` bash
mkdir /mnt/storage
mount /dev/vdb1 /mnt/storage/
```

### 7.设置自动挂载
磁盘被手动挂载之后都必须把挂载信息写入 /etc/fstab 这个文件中，否则下次开机启动时仍然需要重新挂载。系统开机时会主动读取 /etc/fstab 这个文件中的内容，根据文件里面的配置挂载磁盘。这样我们只需要将磁盘的挂载信息写入这个文件中我们就不需要每次开机启动之后手动进行挂载了。

首先通过 blkid 命令将分区的 uuid 查询出来并复制 uuid（往/etc/fstab中追加挂载信息时建议使用uuid）
``` bash
vim /etc/fstab
```
