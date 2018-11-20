# 逻辑卷管理
&ensp;&ensp;&ensp;&ensp;逻辑卷是将一个或多个物理块设备组成一个逻辑设备使用，通过内核中的dm(device mapper)模块实现；逻辑卷的管理工具为LVM2，逻辑卷的实现过程如下，一个或多个块设备标识为物理卷(physical volumes)，然后基于这些物理卷创建卷组(group volumes)，卷组将存储空间分割为相同大小的PE(physical extent)；然后在卷组上创建逻辑卷；逻辑卷可以占用卷组的或部分PE，卷组可以通过增加或移除物理卷调整PE容量，从而实现逻辑卷空间的扩展或收缩，所以LVM相较于传统分区方式更灵活；

一个或多个块设备(Block devices) ---1---> 一个或多个物理卷(physical volumes) ---2----> 卷组(Group volumes) -----3----> 逻辑卷(logical volumes)

1，2，3过程实现的工具和过程下面我们会详细介绍。

&ensp;&ensp;&ensp;&ensp;逻辑卷设备名称为/dev/dm-#，#代表一个数字，用于标识不同的逻辑卷设备。一个逻辑卷设备有两个软链接/dev/mapper/VG_NAME-LV_NAME和/dev/VG_NAME/LV_NAME，其中VG_NAME是逻辑卷所在的卷组名称，LV_NAME是逻辑卷名称；目前CentOS7安装的默认分区方式即会创建一个容量较大的卷组，在此卷组上创建不同的逻辑卷。下面详细介绍创建逻辑卷、扩展逻辑卷、缩减逻辑卷、迁移逻辑卷和逻辑卷快照使用。
## 创建逻辑卷
1. **创建物理卷**

  \# pvcreate device 

   注意：linux逻辑卷分区id为8e，如果使用某磁盘分区而不是一整块磁盘创建逻辑卷，需先修改分区id

  \# pvs 或 # pvdisplaya 查看系统已有的物理卷信息
```bash
使用/dev/sda6和/dev/sda7创建两个物理卷
[[root@centos7 ~]#pvdisplay
  "/dev/sda6" is a new physical volume of "10.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sda6
  VG Name               
  PV Size               10.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               IbX0uQ-PReU-ak2K-aUUc-Yl8R-l7t4-YarrwU
   
  "/dev/sda7" is a new physical volume of "20.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sda7
  VG Name               
  PV Size               20.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               rRQJkU-RRUb-5EyD-2tfu-u556-5XuS-jsyy3h
```
2. **创建卷组**

 使用一个或多个物理卷创建卷组，创建卷组时可以指定PE大小，给出卷组名称，使用物理卷即可

  \# vgcreate [-s UNIT] VG_NAME PV_DEVICE

 查看系统已有的卷组信息

  \# vgs 或 # vgdisplay 
```bash
使用物理卷/dev/sda6, /dev/sda7创建名为testvg的卷组，PE大小使用默认值4M
[root@centos7 ~]#vgcreate testvg /dev/sda{6,7}
  Volume group "testvg" successfully created
[root@centos7 ~]#vgdisplay
  --- Volume group ---
  VG Name               testvg
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               29.99 GiB
  PE Size               4.00 MiB
  Total PE              7678
  Alloc PE / Size       0 / 0   
  Free  PE / Size       7678 / 29.99 GiB
  VG UUID               LRfTvT-uuCi-DckG-HLrB-p7fW-oLdv-dLSkIb
```
3. **创建逻辑卷**

 创建逻辑卷可以指定逻辑卷大小(最大为所在卷组空间)，给出逻辑卷名称，所在卷组即可

  \# lvcreate -L #[mMgGtT] -n LV_NAME VG_NAME

  查看系统已有的逻辑卷信息

  \# lvs 或 # lvdisplay 
```bash
[root@centos7 ~]#lvdisplay
  --- Logical volume ---
  LV Path                /dev/testvg/testlv
  LV Name                testlv
  VG Name                testvg
  LV UUID                KtL3AF-cxn9-kbIK-JXiF-YQpp-fAJC-2CN44s
  LV Write Access        read/write
  LV Creation host, time centos7.localdomain, 2018-10-22 21:38:44 +0800
  LV Status              available
  # open                 0
  LV Size                15.00 GiB
  Current LE             3840
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```
4. **为逻辑卷格式化文件系统**
```bash
[root@centos7 ~]#mkfs.xfs /dev/testvg/testlv
meta-data=/dev/testvg/testlv     isize=512    agcount=4, agsize=983040 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=3932160, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
5. **挂载**
```bash
[root@centos7 ~]#mkdir /mnt/lvm
[root@centos7 ~]#mount /dev/testvg/testlv /mnt/lvm
[root@centos7 ~]#fdisk -l 
Disk /dev/mapper/testvg-testlv: 16.1 GB, 16106127360 bytes, 31457280 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
完成上述过程，逻辑卷就可以使用了。
## 扩展逻辑卷
&ensp;&ensp;&ensp;&ensp;如果逻辑卷所在卷组还有剩余空间，可以直接扩展逻辑卷；如果卷组剩余空间不够，可以先扩展卷组空间，再扩展逻辑卷。逻辑卷空间扩展后，还需要同步扩展逻辑卷的文件系统，这样扩展空间才可以使用；逻辑卷可以在挂载的状态下进行扩展；下面我们从扩展卷组空间为例进行说明：
1. 扩展卷组
  卷组添加物理卷
  \# vgextend VG_NAME PV_DEVICE

[root@centos7 ~]#vgextend testvg /dev/sda8

2. 扩展逻辑卷
  
  \# lvextend  -L [+]#[m|UNIT]|-l +NUMBER[%] /PATH/TO/LV

  指定逻辑卷扩展空间大小4种表示：
   *  -L #[UNIT]:指明扩展后逻辑卷大小
   *  -L +#[UINT]: 指明逻辑卷空间的增加量
   *  -l +NUMBER:指明逻辑卷增加PE数
   *  -l +NUMBER%:指明扩展逻辑卷至卷组空间的百分比
```bash
[root@centos7 ~]#lvextend -l +100%Free /dev/testvg/testlv
  Size of logical volume testvg/testlv changed from 15.00 GiB (3840 extents) to <39.99 GiB (10237 extents).
  Logical volume testvg/testlv successfully resized.
```
3. 扩展文件系统

 \# resize2fs /PATH/TO/LV [size] ，给出目标size; ext系列文件系统

 \# xfs_growfs mount-point [-L size]，给出目标size; xfs文件系统；xfs文件系统不支持缩减
```bash
[root@centos7 ~]#xfs_growfs /mnt/lvm
meta-data=/dev/mapper/testvg-testlv isize=512    agcount=4, agsize=983040 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=3932160, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 3932160 to 10482688
[root@centos7 ~]#df -h
Filesystem                 Size  Used Avail Use% Mounted on
/dev/mapper/testvg-testlv   40G  1.1G   39G   3% /mnt/lvm
```
以上3步操作逻辑卷扩展完成。

扩展逻辑卷也可以使用-r选项，不需要考虑文件系统类型，一步完成逻辑卷和文件系统的同步扩展

 **\# lvcreate -r -L [+]#[m|UNIT]|-l +NUMBER[%] /PATH/TO/LV**

## 缩减逻辑卷
&ensp;&ensp;&ensp;&ensp;不同于在线扩展，逻辑卷缩减时必需在卸载状态进行，同时要做好数据备份；逻辑卷缩减时要先缩减文件系统边界，再缩减逻辑卷物理空间边界，缩减完成后，重新挂载使用即可。
1. 取消逻辑卷挂载
  \# umount MOUNT_POINT
2. 文件系统检查
  \# e2fsck -f /PATH/TO/LV
3. 缩减文件系统边界
  \# resize2fs /PATH/TO/LV size, 给出目标size
4. 缩减逻辑卷
  \# lvreduce  -L [-]Size[m|UNIT] LV , 给出目标size
5. 挂载

&ensp;&ensp;&ensp;&ensp;如果需要同时缩减卷组空间，移除PV，需先将指定PV上LV占用的PE转移至其它还有空闲PE的PV上再执行操作。
 
  \# pvmove PATH/TO/source_PV /PATH/TO/dest_PV
```bash
[root@centos7 ~]#pvmove /dev/sda6 /dev/sda8
  /dev/sda6: Moved: 0.16%
  /dev/sda6: Moved: 53.01%
  /dev/sda6: Moved: 100.00%
```
## 迁移逻辑卷

1. 取消挂载
1. 确认组成逻辑卷的物理卷空间有剩余PE，将要迁移的物理卷上被LV占用的PE迁移至其它空余PV上；
    \# pvmove
2. 禁用逻辑卷# lvchange -an LV_NAME
3. 导出卷组#vgexport VG_NAME
   迁移物理硬盘
4. 导入卷组 # vgimport VG_NAME
5. 启用逻辑卷 # lvchange -ay LV_NAME
6. 挂载LV

## 逻辑卷快照
&ensp;&ensp;&ensp;&ensp;逻辑卷的快照是一个特殊的逻辑卷，快照可以将创建快照时逻辑卷的信息记录下来，当逻辑卷中的数据发生变化，原始数据会保存在快照中，没有发生变化的数据由逻辑卷和快照共享；快照和照片很类似，即把逻辑卷在创建快照时的状态定格在快照中。快照卷的大小取决于做快照时，期望快照的存活时间和原数据的变化量和卷组剩余空间；创建快照后，可以基于快照对数据备份；快照使用完后即时删除。
 1. 创建快照
   \# lvgreate -L #[mMgGtT] -p r -s -n snapshot_LV_name /PATH/TO/original_LV
  * -L #[mMgGtT]：指定快照大小
  * -p r:指定快照卷为只读
  * -s :表明创建的是快照卷
  * -n snapshot_NAME:指定快照名
```bash 
[root@centos7 ~]#lvcreate -s -L 1G -n test_snapshot /dev/testvg/testlv
  Logical volume "test_snapshot" created.
[root@centos7 ~]#blkid
/dev/mapper/testvg-testlv: UUID="02d56a0c-26fc-41ad-b1a5-812e9933cc6a" TYPE="xfs" 
/dev/mapper/testvg-test_snapshot: UUID="02d56a0c-26fc-41ad-b1a5-812e9933cc6a" TYPE="xfs" 
[root@centos7 ~]#lvdisplay
  --- Logical volume ---
  LV Path                /dev/testvg/testlv
  LV Name                testlv
  VG Name                testvg
  LV UUID                KtL3AF-cxn9-kbIK-JXiF-YQpp-fAJC-2CN44s
  LV Write Access        read/write
  LV Creation host, time centos7.localdomain, 2018-10-22 21:38:44 +0800
  LV snapshot status     source of
                         test_snapshot [active]
  LV Status              available
  # open                 1
  LV Size                25.00 GiB
  Current LE             6400
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
   
  --- Logical volume ---
  LV Path                /dev/testvg/test_snapshot
  LV Name                test_snapshot
  VG Name                testvg
  LV UUID                9a5eE5-blD7-UHxt-DsLA-vdok-k9S8-VaPiqa
  LV Write Access        read/write
  LV Creation host, time centos7.localdomain, 2018-10-23 14:42:49 +0800
  LV snapshot status     active destination for testlv
  LV Status              available
  # open                 0
  LV Size                25.00 GiB
  Current LE             6400
  COW-table size         1.00 GiB
  COW-table LE           256
  Allocated to snapshot  0.00%
  Snapshot chunk size    4.00 KiB
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:3
```
 2. 挂载
   \# monut -o ro device mount_point ；挂载后即可访问快照

 对于xfs文件系统，因逻辑卷和快照卷uuid相同，所以挂载快照时需设定忽略uuid

  \# mount -o nouuid,ro device mount_point
```bash
[root@centos7 ~]#mount -o nouuid,ro /dev/testvg/test_snapshot /mnt/snap

逻辑卷中数据发生变化，快照中会保持原始数据；
[root@centos7 ~]#cat /mnt/lvm/issue /mnt/snap/issue
\S
Kernel \r on an \m
add this new line
\S
Kernel \r on an \m

```














