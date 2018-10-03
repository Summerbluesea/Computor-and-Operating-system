# Linux文件系统层级结构及其属性和基本访问权限
  本文详细介绍了linux文件系统层级结构，linux文件属性及其基本访问权限等内容。
## linux文件系统层级结构
  linux系统重要的哲学思想之一：“一切皆文件”。linux几乎将所有资源统统抽象为文件形式，包括硬件设备，这样用户可以用读写文件的形式实现对硬件设备的访问。
  linux使用层级结构将这些文件有序的组织，便于用户对文件的访问；linux文件系统层级结构像一颗倒置的大树，最顶端是根目录/，/下又划分了不同的次目录用于存放不同类型文件；FHS（Filesystem Hierarchy Standard）规定了linux文件系统层级结构标准，下面概略介绍FHS标准层级结构有哪些，分别存放什么类型文件。详细请参考[filesystem hierarchy standard](http://www.pathname.com/fhs/)。

  /：一切文件的最顶层组织者，其下包含系统启动、还原、恢复和修复必须的文件；这些文件按不同性质和功能存放在根下的二级目录/boot, /bin, /sbin, /etc, /dev, /lib(/lib64), /var,  /srv, /usr,  /proc, /sys, ...

  /boot:该目录包括了系统启动时所需的一切文件，除了系统刚启动加载时不需要的配置文件和系统安装指南；所以/boot下存放的文件是在kernel还未开始执行用户模式前使用的；

  /bin:所有用户可用的基本命令程序文件

  /sbin:系统管理的程序文件

  /etc :存放程序的配置文件，/etc下的文件必须是静态的且不能为二进制文件

  /dev:存放设备文件和特殊文件

  /lib:基本的共享库文件和kernel模块，为系统启动或根文件系统上的程序（/bin, /sbin） 提供共享库，以及为内核提供内核模块；其下的主要子目录如下：
   - libc.so.*: 动态链接的c库
   + ld*： 运行时连接器/加载器
   * modual: 存放内核模块的目录

  /lib64：64位系统特有的存放共享库的路径

  /home:普通用户的家目录存放路径

  /root:管理员的家目录，可选的

  /media：其子目录是便携式设备的挂载点

  /mnt：其它文件系统的临时挂载点

  /var: 存放经常发生变化的数据，其子目录有/cache, /log, /lib, ...

  /tmp: 为那些会产生临时文件的程序提供临时文件存放路径

  /srv:系统服务数据

  /usr: 是根文件系统的第二大部分，存放全局共享只读文件，其子目录有：
  -  /usr/bin：大部分用户命令，/bin是/usr/bin目录的软连接，使用`ll -di /bin`查看如下所示

    ` [root@centos7 ~]# ll -di /bin
      228427 lrwxrwxrwx. 1 root root 7 Sep 19 20:50 /bin -> usr/bin `

  +  /usr/sbin:不是系统启动时必须的程序，系统管理的程序文件， /sbin是/usr/sbin目录的软连接，使用`ll -di /sbin`查看如下所示
 
     ` [root@centos7 ~]# ll -di /sbin
     85 lrwxrwxrwx. 1 root root 8 Sep 19 20:50 /sbin -> usr/sbin `

  *  /usr/lib, /usr/lib64/: 分别是/lib, /lib64目录的软链接

  *  /usr/share:命令手册和自带文档等文件架构特有文件储存路径
  -  /usr/local hierarchy：存放系统管理员安装的本地文件/第三方程序

  /proc: 基于内存的虚拟文件系统，将内核和进程运行的相关信息抽象为文件系统格式，便于用户访问，多为内核参数；

  /sys：/sysfs提供了一种比/proc更为理想的访问内核数据的途径；为管理linux设备提供了一种统一模式的接口

以上简要介绍了linux文件系统FHS标准层级目录结构。

## linux系统的文件属性
linux系统文件的属性包括文件名、文件类型、文件字节数、文件的访问权限、属主、属组、inode、链接数、timestamps等信息，这些文件的属性称为文件的元数据，文件内容称为文件的数据，文件的元数据和数据存放在磁盘不同的位置，用户要访问某一文件，首先通过文件路径映射找到目录文件，在目录文件中读取与文件名相对应的inode节点号，在同一分区中，inode节点号与文件名对应；通过inode节点号找到文件inode中存储的文件数据磁盘位置，然后找到位置访问文件数据。
### linux文件的文件名
在inux中，文件名存储在文件所在目录下的目录文件中。
### linux系统的文件类型
对于linux而言，文件是存储空间上的一段流式数据；文件系统是层级组织文件的结构，linux系统上的文件类型及其代表字符如下所示：

 - -：常规文件
 + d: directory, 目录文件，文件路径映射
 * b: block device，块设备文件，支持以“block”为单位随机访问
 - c: charactor device, 字符设备文件，支持以“charactor”为单位线性访问
 + l: symbolic link, 符号链接文件
 * p: pipe，命名管道
 - s:socket，套接字文件

使用命令`# file FILENAME`可查看文件类型

### linux文件的访问权限以及属主、属组
用户对于文件的访问模式（mode）有三种，分别为 可读readable(r), 可写writable(w), 可执行excutable(x)；这三种访问模式在文件和目录文件上又有不同的体现，下面将分别介绍：

对于文件而言：
 - readable :可获取文件内容
 + writable:可修改文件内容
 * excutable:可将此文件发起运行为进程

对于目录文件而言：
 - readable :可使用ls命令获取目录文件内容列表
 + writable:可新建或者删除目录文件列表内容
 * excutable:可cd至此目录中，且可使用ls -l 获取所有目录文件列表中文件的详细属性信息


文件的访问权限根据其ownership划分为3个类别，分别为属主（user）, 属组（group）, 其它（other）；属主即为文件的所有者，属组为文件所属组，既不是文件所有者也不是文件所属者即为其它；某一用户发起的进程对文件的访问权限应用模型如下： 进程的发起者与文件的属主是否相同；如果相同，则应用属主权限；否则，检查进程的发起者是否属于文件的属组，如果是，则应用属组权限；否则，就只能应用other权限。

文件访问权限的管理命令:
- `# chmod` 修改文件的访问模式；使用格式如下：

  ` 1. # chmod [options]...MODE,[MODE] FILE

      MODE表示法：
        u=,g=,o=,a=
        u+，u-，g+，g-，o+，o-，a+，a-

  ` 2. # chmod [options]...OCTAL-MODE FILE`

   `3. # chmod --reference=REFILE file `

+ `# chown`修改文件属主；使用格式如下：

    `# chown [options]..[owner][:group] FILE`或
    `# chown --reference=REFILE FILE` 

* `# chgrp`修改文件属组；因使用chown命令即可修改文件属组，所以chgrp很少使用，这里不再介绍其使用格式。

### linux文件的inode
linux文件系统在硬盘分区格式化时，每一个分区都划分出存储目录文件、inode table和数据的区域；一个文件的存储由一个目录项、inode和一定数量的数据块完成；其中，目录项中存储文件名和与文件名对应的inode节点号，inode节点号（文件索引节点）对应于文件的inode的存放位置；inode中存储了文件元数据除文件名的所有信息和数据块指针（pointer），通过pointer找到文件数据存储位置；数据块中存储文件内容。可使用命令`ls -i`查看文件的inode节点号，使用命令`stat`查看文件inode信息；

`[root@centos7 ~]# ls -i /etc/issue
 67205655 /etc/issue`

67205655就是文件/etc/issue的inode节点号

```bash[root@centos7 ~]# stat /etc/issue

  File: ‘/etc/issue’
  Size: 23        	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 67205655    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: system_u:object_r:etc_t:s0
Access: 2018-10-02 10:01:14.255010893 +0800
Modify: 2018-04-29 00:35:55.000000000 +0800
Change: 2018-09-19 20:50:30.234987303 +0800
 Birth: -
```

### linux文件的链接数

通过前面叙述，我们已经了解到linux中文件的存储机制；所以在linux中，inode节点号是文件的唯一标识而非文件名；inode中links表示的链接数是硬链接（hard link），可以理解为有多少个文件名指向这个inode节点号；所以，硬链接类似于文件的别名，可使用`ln`命令创建文件的硬链接：
```bash
[root@centos7 ~]# echo "this is the original" > /data/file1
[root@centos7 ~]# ll -i /data/file1
3168 -rw-r--r--. 1 root root 21 Oct  2 20:22 /data/file1
[root@centos7 ~]# ln /data/file1 /data/file1.link
[root@centos7 ~]# ll -i /data/file1 /data/file1.link
3168 -rw-r--r--. 2 root root 21 Oct  2 20:22 /data/file1
3168 -rw-r--r--. 2 root root 21 Oct  2 20:22 /data/file1.link
```
创建文件的硬链接，仅文件的目录项中更新，新增了一个指向相同inode节点号的文件信息，文件的inode信息中links数增加一个；


文件的软链接（soft link or symbolic link）是一个独立的文件，软链接文件的内容为访问该文件的路径；所以软链接文件有自己的目录项、inode和数据块。程序访问软链接文件获取文件路径，根据路径访问文件数据，所以软链接类似于windows系统中的快捷方式。可使用`ln -s`命令创建文件的软链接：
```bash
[root@centos7 ~]# ln -s /data/file1 /tmp/file1.link3
[root@centos7 ~]# ll -i /data/file1 /data/file1.link /tmp/file1.link3
    3168 -rw-r--r--. 2 root root 21 Oct  2 20:22 /data/file1
    3168 -rw-r--r--. 2 root root 21 Oct  2 20:22 /data/file1.link
67163970 lrwxrwxrwx. 1 root root 11 Oct  2 20:40 /tmp/file1.link3 -> /data/file1
```
由上所述，文件硬链接和软连接的主要区别有如下几个方面

  1. 硬链接与文件是同一个文件，可以理解为文件的一个别名，删除原文件还可以访问文件数据；软链接文件是独立于原文件的一个文件，是文件的另一种访问方式，删除原文件后软链接文件仍存在，但无法访问，成为一个死链接；若重新创建一个与原文件同名的文件，则软链接文件又可以访问了；
  ```bash
  [root@centos7 ~]# rm /data/file1
rm: remove regular file ‘/data/file1’? y
[root@centos7 ~]# cat /data/file1.link
this is the original
[root@centos7 ~]# cat /tmp/file1.link3
cat: /tmp/file1.link3: No such file or directory
[root@centos7 ~]# ll -i /tmp/file1.link3
67163970 lrwxrwxrwx. 1 root root 11 Oct  2 20:40 /tmp/file1.link3 -> /data/file1

[root@centos7 ~]# rm /data/file1
rm: remove regular file ‘/data/file1’? y
[root@centos7 ~]# cat /data/file1.link
this is the original
[root@centos7 ~]# cat /tmp/file1.link3
cat: /tmp/file1.link3: No such file or directory
[root@centos7 ~]# ll -i /tmp/file1.link3
67163970 lrwxrwxrwx. 1 root root 11 Oct  2 20:40 /tmp/file1.link3 -> /data/file1

[root@centos7 ~]# cat /data/file1 /data/file1.link /tmp/file1.link3
this is the second version
this is the original
this is the second version
```
  2. 因为不同分区有自己的Inode节点号，所以硬链接不能跨分区创建，而软链接可以。
  3. 不能对目录文件创建硬链接，而软链接可以。
  4. 硬链接文件的inode号与文件相同，软链接则不同



### linux文件的timestamps

文件的三个时间标签timestamps
 - atime，access time, 最近一次访问的时间，
 + ctime，change time 文件属性更改时间，最近一次元数据的更改时间
 * mtime, modify time  最近一次文件内容更改的时间

 可使用`stat`命令查看文件的timestamps
```bash 
[root@centos7 ~]# stat /data/file1
  File: ‘/data/file1’
  Size: 27        	Blocks: 8          IO Block: 4096   regular file
Device: 803h/2051d	Inode: 3169        Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: unconfined_u:object_r:etc_runtime_t:s0
Access: 2018-10-02 20:57:18.827529864 +0800
Modify: 2018-10-02 20:54:56.413532909 +0800
Change: 2018-10-02 20:54:56.413532909 +0800
 Birth: -
```      
以上即为本篇所有内容，文中介绍了linux文件系统层级结构，linux中文件的各种属性和访问权限等内容。