---
title: create LVM disk
comments: true
date: 2017-07-19 16:24:04
updated: 2017-07-19 16:24:04
tags:
- LVM
- DISK
- LINUX
categories:
- LINUX
- LVM
---
### 创建物理分区
#### 查看是否有未分区的磁盘
```
[root@srm-fs ~]# fdisk -l
Disk /dev/xvda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000d6814
    Device Boot      Start         End      Blocks   Id  System
/dev/xvda1               1         523     4194304   82  Linux swap / Solaris
Partition 1 does not end on cylinder boundary.
/dev/xvda2   *         523        5222    37747712   83  Linux
Disk /dev/xvdb: 536.9 GB, 536870912000 bytes
255 heads, 63 sectors/track, 65270 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
Disk /dev/xvdc: 536.9 GB, 536870912000 bytes
255 heads, 63 sectors/track, 65270 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```
发现有两个未使用的磁盘，这里以xvdb为例
<!-- more -->
#### 在磁盘xvdb上创建xvdb1分区
```
[root@srm-fs ~]# fdisk /dev/xvdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xf72f347d.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.
Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)
WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-65270, default 1):
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-65270, default 65270):
Using default value 65270
添加磁盘系统表id为8e(LVM的标识)
Command (m for help): t
Selected partition 1
Hex code (type L to list codes): 8e
Changed system type of partition 1 to 8e (Linux LVM)
保存分区表
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
```
### 将分区划分为物理卷，主要是添加LVM属性信息并划分PE存储单元.
```
[root@srm-fs ~]# pvcreate /dev/xvdb1
  Physical volume "/dev/xvdb1" successfully created
[root@srm-fs ~]# pvs
  PV         VG   Fmt  Attr PSize   PFree  
  /dev/xvdb1      lvm2 a--  499.99g 499.99g
[root@srm-fs ~]# pvdisplay
  "/dev/xvdb1" is a new physical volume of "499.99 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/xvdb1
  VG Name               
  PV Size               499.99 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               5KBToI-5jzg-P0ra-kdZO-3bt2-MRS2-Hbbjk3
```
### 创建卷组vg-atm, 并将分区xvdb1加入卷组
其中PE为卷组的最小存储单元，可通过-s命令修改
```
[root@srm-fs ~]# vgcreate vg-atm /dev/xvdb1
  Volume group "vg-atm" successfully created
[root@srm-fs ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree  
  vg-atm   1   0   0 wz--n- 499.99g 499.99g
[root@srm-fs ~]# vgdisplay
  --- Volume group ---
  VG Name               vg-atm
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               499.99 GiB
  PE Size               4.00 MiB
  Total PE              127998
  Alloc PE / Size       0 / 0   
  Free  PE / Size       127998 / 499.99 GiB
  VG UUID               8Enzk5-pm35-z6OQ-ee2v-RW4X-vp1n-00Z2Ua
```
#### 从vg-atm卷组创建逻辑分区lv-atm1
```
[root@srm-fs ~]# lvcreate -l 127998 -n lv-atm1 vg-atm
  Logical volume "lv-atm1" created
其中，指定创建的逻辑分区大小有两种方式:
-L 后面指定大小(e.g. 499G)不能超过卷组的free size，可通过vgdisplay查看
-l 后面指定PE大小(e.g. 127998)不能超过卷组的free PE大小，可通过vgdisplay查看
[root@srm-fs ~]# lvs
  LV      VG     Attr       LSize   Pool Origin Data%  Move Log Cpy%Sync Convert
  lv-atm1 vg-atm -wi-a----- 499.99g

[root@srm-fs ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/vg-atm/lv-atm1
  LV Name                lv-atm1
  VG Name                vg-atm
  LV UUID                U8nc8o-aa2p-NeNw-JMie-mjoh-eRfo-wKdQuS
  LV Write Access        read/write
  LV Creation host, time srm-fs, 2017-06-07 10:42:57 +0800
  LV Status              available
  # open                 0
  LV Size                499.99 GiB
  Current LE             127998
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```
### 在逻辑分区lv-atm上创建文件系统
```
[root@srm-fs ~]# mkfs.ext4 /dev/vg-atm/lv-atm1
mke2fs 1.41.12 (17-May-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
32768000 inodes, 131069952 blocks
6553497 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
4000 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
	102400000
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
This filesystem will be automatically checked every 25 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```
### 挂载文件系统
```
[root@srm-fs ~]# mount /dev/vg-atm/lv-atm1 /u01
[root@srm-fs ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/xvda2                     36G  2.4G   32G   7% /
tmpfs                         487M     0  487M   0% /dev/shm
/dev/mapper/vg--atm-lv--atm1  493G  198M  467G   1% /u01
[root@srm-fs ~]# mount | grep u01
/dev/mapper/vg--atm-lv--atm1 on /u01 type ext4
创建好之后,会在/dev/mapper/生成一个软连接名字为”卷组-逻辑卷”从上面的命令也可以看出来。
[root@srm-fs ~]# ll /dev/vg-atm/lv-atm1
lrwxrwxrwx 1 root root 7 Jun  7 13:32 /dev/vg-atm/lv-atm1 -> ../dm-0
```
### 配置开机自动挂载
```
查看新增分区的UUID
[root@srm-fs ~]# blkid
/dev/xvda1: UUID="25ec3bdb-ba24-4561-bcdc-802edf42b85f" TYPE="swap"
/dev/xvda2: UUID="1a1ce4de-e56a-4e1f-864d-31b7d9dfb547" TYPE="ext4"
/dev/xvdb1: UUID="5KBToI-5jzg-P0ra-kdZO-3bt2-MRS2-Hbbjk3" TYPE="LVM2_member"
/dev/mapper/vg--atm-lv--atm1: UUID="eb80318b-480f-486c-8b7c-6093974139f8" TYPE="ext4"
编辑/etc/fstab,添加如下内容
UUID=eb80318b-480f-486c-8b7c-6093974139f8" TYPE="ext4  /u01    ext4   defaults   1 1
```
### 添加新的物理分区至卷组vg-atm
同样需要先创建pv，具体步骤省略
```
vgextend vg-atm /dev/xvdc1
```
### 扩展逻辑分区lv-atm
```
lvextend -L +500M /dev/vg-atm/lv-atm1
路径为逻辑分区的路径，可通过lvdisplay查看
```
