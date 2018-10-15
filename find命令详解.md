# find命令详解
find命令是一个实时文件查找工具，通过遍历指定搜索起始路径下的所有文件查找符合给定条件的文件，还可以通过固定命令格式或结合xargs命令对符合条件的文件批量处理；查找条件可以指定文件的各种属性，如文件名、类型、大小、从属关系、访问权限、时间戳等，还可以组合使用；用法如下:

   #find [options] [指定查找路径][查找条件1 组合逻辑 查找条件2...][处理动作]

若未指定路径，find默认在当前目录下查找。也可以指定目录下的搜索层级，下面详述各查找条件和处理动作的使用。
## 查找条件
各查找条件可以通过组合逻辑组合使用，以精确查找文件；组合条件有以下三种：
 + 与， 用符号-a表示，是find的默认组合逻辑，两条件必须同时满足；
 - 或，用符号-o表示，任一条件满足即可；
 * 非，用-not或!表示，查找条件取反
### 根据文件名查找

    #find /DIR -name "PATTERN" 或 #find /DIR -iname "PATTERN"

  + 使用-iname忽略字符大小写；
  - "PATTERN"支持glob风格通配符;
      +  \*  匹配任意长度的任意字符
      - ? 匹配任意单个字符
      * [] 匹配指定范围内的任意单个字符
      + [^] 匹配指定范围外的任意单个字符
搜索/data目录下文件名以.sh结尾的文件
```bash
[root@centos7 ~]#find /data -name "*.sh"  
/data/scripts/sysinfo_07.sh
/data/scripts/createsscript_07.sh
/data/scripts/backup.sh
/data/scripts/ip_07.sh
/data/color.sh
```
指定搜索目录层级可以使用-maxdepth LEVEL(最大搜索层级,LEVEL用数字替代)，-mindepth LEVEL(最小搜索层级)；上面的例子如果指定最大搜索层级为1，只搜索/data目录下以.sh结尾的文件，不会深入到/data下的子目录/data/scripts搜索，结果如下：
```bash
[root@centos7 ~]#find /data -maxdepth 1  -name "*.sh"  
/data/color.sh
```
### 根据文件类型查找
 #find /DIR -type TYPE 
     f:普通文件
     d:目录文件
     l:符号链接文件
     b:块设备文件
     c:字符设备文件
     s:套接字文件
     p:管道文件
```bash
查找/etc目录下的符号链接文件，指定目录搜索层级为1
[root@centos7 ~]#find /etc  -maxdepth 1 -type l -ls
67160132    0 lrwxrwxrwx   1 root     root           17 Sep 19 20:50 /etc/mtab -> /proc/self/mounts
67163331    0 lrwxrwxrwx   1 root     root           11 Sep 19 20:50 /etc/init.d -> rc.d/init.d
67163334    0 lrwxrwxrwx   1 root     root           10 Sep 19 20:50 /etc/rc0.d -> rc.d/rc0.d
67163335    0 lrwxrwxrwx   1 root     root           10 Sep 19 20:50 /etc/rc1.d -> rc.d/rc1.d
67163336    0 lrwxrwxrwx   1 root     root           10 Sep 19 20:50 /etc/rc2.d -> rc.d/rc2.d
```
### 根据文件从属关系查找
可以指定文件的属主、属组或id号查询
  + #find /DIR -user USERNAME  或 #find /DIR -uid UID ；指定查找文件属主
  - #find /DIR -group GRPNAME；或#find /DIR -gid GID；指定查找文件属组
  - #find /DIR -nouser；查找没有属主的文件
  + #find /DIR -nogroup；查找没有属组的文件
查找文件系统没有属主或没有属组的文件并使用-ls命令列出长格式信息，需注意的是组合条件需用()，否则命令执行不正确
```bash
[root@centos7 ~]#find / \( -nouser -o -nogroup \) -ls
find: ‘/proc/2710/task/2710/fd/6’: No such file or directory
find: ‘/proc/2710/task/2710/fdinfo/6’: No such file or directory
find: ‘/proc/2710/fd/5’: No such file or directory
find: ‘/proc/2710/fdinfo/5’: No such file or directory
67541465    4 -rw-r--r--   1 4001     4001           12 Oct  7 21:01 /etc/file.gen
703856    0 drwxrwxr-x   2 4001     4001            6 Oct  7 12:06 /tmp/test1
67163581    0 -rw-rw-r--   1 4001     4001            0 Oct  7 12:07 /tmp/file.gen
``` 
### 根据文件大小查找
  #find /DIR -size [+|-]#UNIT
  + UNIT: k, M，G；指定单位
  * #UNIT:(#-1, #]；例如指定找大小为10k的文件，匹配的文件是大于9k且小于等于10k文件。
  * -#UNIT:(0, #-1]；例如指定查找小于10k的文件，匹配所有大于0k小于等于9k的文件
  * +UNIT:(#,无穷大)；例如指定查找大于10k的文件，匹配所有大于10k的文件
```bash
[root@centos7 ~]#ll /data
-rw-------.  1 root  root  26393 Oct  7 15:27 messages
-rw-------   1 root  root  26842 Oct 13 16:33 messages.1
-rw-------.  1 root  root  27787 Oct  7 15:31 messages.2
-rw-------.  1 root  root  27511 Oct  7 15:33 messages.3
查找/data目录下文件大小为27k的文件，指定目录搜索层级为1
[root@centos7 ~]#find /data -maxdepth 1 -size 27k -ls
    71   28 -rw-------   1 root     root        27511 Oct  7 15:33 /data/messages.3
 20365   28 -rw-------   1 root     root        26842 Oct 13 16:33 /data/messages.1
 [root@centos7 ~]#find /data  -maxdepth 1 -size 27k -exec ls -lh {} \;
-rw-------. 1 root root 27K Oct  7 15:33 /data/messages.3
-rw------- 1 root root 27K Oct 13 16:33 /data/messages.1

 查找/data目录下大于27k的文件，指定目录搜索层级为1
 [root@centos7 ~]#find /data -maxdepth 1 -size +27k -ls
    70   28 -rw-------   1 root     root        27787 Oct  7 15:31 /data/messages.2
[root@centos7 ~]#find /data  -maxdepth 1 -size +27k -exec ls -lh {} \;
-rw-------. 1 root root 28K Oct  7 15:31 /data/messages.2

查找/data目录下小于于27k的文件，指定目录搜索层级为1
[root@centos7 ~]#find /data -maxdepth 1 -size -27k -ls
 67   28 -rw-------   1 root     root        26393 Oct  7 15:27 /data/messages
[root@centos7 ~]#find /data \(-maxdepth 1 -size -27k\) -exec ls -lh {} \;
-rw-------.  1 root  root   26K Oct  7 15:27 messages
```
### 根据文件时间戳查找
以天为单位

  + #find /DIR -atime #:距离上一次访问时间#天，(#-1,#]
  - #find /DIR -atime +#:距离上一次访问时间超过#天，(oo，#-1]
  * #find /DIR -atime -#:距离上一次访问时间小于#天，(#，0)
也可以指定-mtime, -ctime或以分钟为单位-amin, -mmin, -cmin
```bash
查找/etc目录下距离上一次修改时间为1天的文件
[root@centos7 ~]#find /etc/ -mtime 1 -ls
33571878    4 -rw-r--r--   1 root     root          312 Oct 12 16:16 /etc/default/grub~
33898218    0 drwxr-xr-x   5 root     root           81 Oct 12 14:44 /etc/selinux
33579035    4 -rw-r--r--   1 root     root          546 Oct 12 14:44 /etc/selinux/config
查找/etc目录下距离上一次修改时间小于1天的文件
[root@centos7 ~]#find /etc/ -mtime -1 -ls
67160129   12 drwxr-xr-x  76 root     root         8192 Oct 13 14:18 /etc/
67160133    4 -rw-r--r--   1 root     root          127 Oct 13 14:18 /etc/resolv.conf
33555658    0 drwxr-xr-x   2 root     root           86 Oct 12 19:28 /etc/default
33579033    4 -rw-r--r--   1 root     root          235 Oct 12 19:24 /etc/default/grubr
33571877    4 -rw-r--r--   1 root     root          249 Oct 12 19:28 /etc/default/grub
67547684    4 -rw-r--r--   1 root     root         1863 Oct 13 09:44 /etc/profile
228424    4 drwxr-xr-x   2 root     root         4096 Oct 13 09:46 /etc/profile.d
524911    4 -rw-r--r--   1 root     root           14 Oct 13 09:07 /etc/tuned/active_profile
524796    4 -rw-r--r--   1 root     root            5 Oct 13 09:07 /etc/tuned/profile_mode
查找/etc目录下距离上一次修改时间大于1天的文件
67541465    4 -rw-r--r--   1 4001     4001           12 Oct  7 21:01 /etc/file.gen
```
### 根据文件访问权限查找
   #find -perm [/|-]mode 
+ mode:精确匹配给定权限
- /mode:任何一类用户的任何一位权限满足条件即可，至少有一类用户的一位权限位满足；9位权限之间是“或”关系
* -mode：每一类用户的每一位权限同时都满足条件；9位权限之间是“与”关系

 查找文件系统上没有属主或属组且最近一周被访问过的文件：     
```bash
[root@centos7 ~]#find / \( -nouser -o -nogroup -a -atime -7 \) -ls
 69    4 -rw-rw-r--   1 4016     4016         1354 Oct 13 17:36 /data/user1.txt
```
至少有一类用户没有执行权限：
```bash
[root@centos7 ~]#find /etc -not -perm -111 -ls
548770   12 -rw-------   1 root     root        12034 Apr 12  2018 /etc/selinux/targeted/active/modules/100/sssd/hll
34036510    0 drwx------   2 root     root           44 Sep 19 20:51 /etc/selinux/targeted/active/modules/100/staff
67547685 -rw-r-xr-- 1 root root 0 Oct 13 17:57 /etc/f1
67547689 -rw-r--r-x 1 root root 0 Oct 13 17:57 /etc/f2
```
所有用户都有执行权限，且其它用户有写权限；
```bash
[root@centos7 ~]#find /etc/init.d  -perm -113 -ls
67163331    0 lrwxrwxrwx   1 root     root           11 Sep 19 20:50 /etc/init.d -> rc.d/init.d
```
## 处理动作
+ -print：默认动作，查找到的内容输出至标准输出
- -ls:输出文件的详细信息
* -delete:删除查找到的文件
+ -fls /PATH/TO/SOMEFILE:把查找到的文件的详细信息存到指定文件
- -ok COMMAND {} \; :固定格式，对查找到的所有文件执行由COMMAND表示的命令
* -exec COMMAND {} \; :对查找到的所有文件执行由COMMAND表示的命令，不再一一确认是否执行此命令
将查找到文件复制并重命名为以.bak为后缀的文件

[root@centos7 data]#find /data -maxdepth 1  -name "*.sh"  -exec cp {} {}.bak \;
### xargs命令
xargs命令用于读入标准输入的多个以分割符（默认为空格）分隔的参数传递给命令，每次传递一个，命令执行后再传递下一个，可以避免因参数过多命令执行失败的情况。
常结合find命令使用,将find查找到的文件一个一个的传递给后面命令，作为后面命令的参数：

    #find | xargs COMMAND