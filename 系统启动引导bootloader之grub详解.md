# CentOS系统启动引导bootloader之grub详解

GRUB(GRand Unified Bootloader)是CentOS系统启动引导程序；CentOS5和CentOS6安装的grub程序版本为grub-0.97，称为grub legacy系列；CentOS7使用的grub程序版本grub-2.0，grub-2.0是完全不同于grub legacy；

grub legacy在引导系统启动时分为3个阶段，相关的文件存放在/boot/grub目录下

```bash
[root@centos6 ~]#ls /boot/grub/
device.map     fat_stage1_5  grub.conf         jfs_stage1_5    reiserfs_stage1_5  stage2         vstafs_stage1_5
e2fs_stage1_5  ffs_stage1_5  iso9660_stage1_5  minix_stage1_5  stage1             ufs2_stage1_5  xfs_stage1_5

grub stage1的文件在系统安装时由安装程序写入磁盘的MBR(前446个字节)，其作用是帮助grub找到stage2的程序；
grub stage1_5程序是加载stage2所在分区文件系统驱动，这样下一步才可以访问stage2程序文件
grub stage2程序文件放在磁盘上的/boot目录下；通常将/boot做一个独立分区

```bash
[root@centos6 ~]#ll /boot
total 40420
-rw-r--r--. 1 root root   108282 Jun 20 05:29 config-2.6.32-754.el6.x86_64
drwxr-xr-x. 2 root root     4096 Nov 11 19:40 grub
-rw-------. 1 root root 18502269 Nov  8 11:34 initramfs-2.6.32-754.el6.x86_64.img
-rw-r--r--. 1 root root   216063 Jun 20 05:30 symvers-2.6.32-754.el6.x86_64.gz
-rw-r--r--. 1 root root  2652834 Jun 20 05:29 System.map-2.6.32-754.el6.x86_64
-rwxr-xr-x. 1 root root  4315504 Jun 20 05:29 vmlinuz-2.6.32-754.el6.x86_64
``` 
**grub legacy功用**
```bash
1. 提供启动菜单, 并提供交互式接口供用户编辑和选择
    e:编辑模式, 用于编辑菜单
    c: 命令行模式, grub 命令行, 交互式接口;
2. 加载用户选择的内核或操作系统
    允许用户输入参数并传递给内核
    可隐藏此菜单(grub菜单可在启动时按任意键唤醒)
3. 为菜单提供了保护机制
    为编辑菜单进行认证
    为启用内核或操作系统进行认证
```
**grub如何识别/boot所在分区设备**
```bash
 grub中磁盘设备表示为hd(#,#)
  第一个#：表示第几块磁盘
  第二个#：表示第#块磁盘的第#分区
grub在引导系统启动时，到stage2阶段，读取/boot/grub/grub.conf配置文件并提供启动菜单供用户选择和编辑，用户也可以进入grub的命令行模式手动编辑启动系统，要为grub指明/boot所在磁盘分区(hd#,#), 要启动内核vmlinuz文件位置和initialramfs文件位置。

1. 手动在grub命令行接口启动系统:
      grub> root (hd#,#)
      grub> kernel /vmlinuz-VERSION-RELEASE ro root=/dev/DEVICE(此处为真正根文件系统所在分区)
      grub> initrd /initramfs-VERSION-RELEASE.img 
      grub> boot 

  如果/boot不是独立分区，/boot在rootfs系统，则grub的根目录直接写/boot即可
      grub> root /boot/
      grub> kernel /boot/vmlinuz-VERSION-RELEASE ro root=/dev/DEVICE(此处为真正根文件系统所在分区)
      grub> initrd /boot/initramfs-VERSION-RELEASE.img 
      grub> boot

2. grub的配置文件: /boot/grub/grub.conf 
    配置项:
      default=#:设定默认启动的菜单项编号; 菜单项(title)从0开始;
      timeout=#:指定菜单项等待用户选择的时长;
      splashimage=(hd#,#)/splash.xpm.gz; 指明grub菜单背景图片文件路径;
      hiddenmenu:隐藏菜单
      password [--md5] STRING:菜单编辑认证;
      title TITLE:定义菜单项"标题"
          root (hd#,#) : grub查找stage2及kernel文件所在设备分区(/boot/), 为grub的"根";
          kernel /vmlinuz_file ro root=/dev/DEVICE [arguments]:启动内核,设定内核参数;
          initrd /initramfs_file:内核匹配的ramfs文件;
          password [--md5] STRING:启动选定内核或操作系统时进行认证;
```
**grub安装**
```bash
1. 使用grub-install命令安装grub,需要指明grub的根目录(boot的父目录)和操作系统所在磁盘
   # grub-install --root-directory=ROOT /dev/disk

2. 进入grub命令行安装，依赖于/boot/grub/目录下grub各阶段文件，如果这些文件不存在，则不能使用此方法安装grub
   # grub
   grub> root(hd0,0)
   grub> setup (hd0)
   grub> boot
```
**grub破坏修复**
```bash
1. MBR的bootloader(grub stage1)破坏修复
   方法一：直接在命令行安装grub
   方法二：如果重启系统，连接光盘，选择关盘引导启动进入救援模式
    (1)切根 # chroot /mnt/sysimage
    (2) 安装grub # grub-install --root-directory=/ /dev/disk
    (3) 救援模式重启系统
2. /boot目录下文件丢失(grub stage2丢失)，包括grub各文件和vmlinuz文件、initramfs文件，重启系统修复
   连接光盘，重启系统，选择关盘引导启动进入救援模式
    (1)切根 # chroot /mnt/sysimage
    (2) 因vmliuz和initramfs文件丢失，需安装kernel
       # mount /dev/sr0 /media/cdrom
       # rpm -ivh /media/cdrom/Packages/kernel-VERSION-RELEASE --force
    
    (3) 安装grub # grub-install --root-directory=/ /dev/disk
    (4) 编辑grub配置文件
        /boot/grub/grub.conf
        default=0
        timeout=5
        title CentOS 6 
            root (hd#,#) 此处为/boot所在硬盘分区，以数字0开始
            kernel /vmlinuz-VERSION-release  ro root=/dev/DEVICE(此处为/所在磁盘分区)
            initrd /initramfs-VERSION-release 
    (5) 救援模式重启系统
```
**新建硬盘实现引导系统启动功能**
```bash
1.新建硬盘，提供单独运行的bash系统
   (1)新添加硬盘
   (2) 分区 /boot, / , swap
   (3)格式化文件系统
   (4)创建临时目录/mnt/boot，挂载/boot分区至临时目录
   (5)为新硬盘安装grub 
      # grub-install --root-directory=/mnt /dev/sdb
   (6) cp 内核文件vmlinuz, initramfs文件至/mnt/boot 
   (7)编辑grub配置文件
      default=0
      timeout=0
      title CentOS 6 express
            root hd(0,0)
            kernel /vmlinuz ro root=/dev/DEVICE selinux=0 inti=/bin/bash
            initrd /initramfs.img
    (8)创建临时目录/mnt/sysroot; 挂载根分区至临时目录； 创建rootfs下目录文件bin, sbin，....
    (9)复制bash等命令和依赖库文件至相应目录
    (10)这块硬盘可以做引导系统启动硬盘
