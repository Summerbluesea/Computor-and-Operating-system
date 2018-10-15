# linux上特殊的文件访问权限总结
上一篇我们介绍了linux系统上基本的文件访问权限，也总结了进程的安全上下文——文件访问权限的应用模型：
当用户运行某一程序文件时，该用户即为进程的发起者，进程的发起者以用户的身份访问文件，首先判断进程发起者是否为文件的属主，如果是，则应用属主权限；
否则，判断进程的发起者是否属于文件的属组，如果是，则应用属组的权限；否则，应用other权限。

这一篇内容将介绍linux上特殊的文件访问权限及其管理应用，这些特殊权限包括SUID, SGID, STICKY 和facl（file access control lists）

## 特殊权限之SUID
默认情况下，用户运行某一程序文件，该用户即为进程的发起者，进程的发起者以用户的身份访问文件；
如果此程序文件有SUID权限，则进程以程序文件属主的身份运行
```bash
[root@centos7 ~]# ll /etc/shadow
----------. 1 root root 1284 Oct  3 11:49 /etc/shadow
[root@centos7 ~]# ll /bin/cat
-rwxr-xr-x. 1 root root 54080 Apr 11 12:35 /bin/cat
[root@centos7 ~]# cat /etc/shadow
root:$6$f6RAFKqJ1Bnk0k0m$v1LyCMQr8J0QBFKB15IhpjWZ6LbnxrorxmEmtxlhFzrcaRQNnwQY0Bxp9qW1cU6pSQYRLDI6lOMLEifh2ozDv0::0:99999:7:::
bin:*:17632:0:99999:7:::
daemon:*:17632:0:99999:7:::
adm:*:17632:0:99999:7:::
lp:*:17632:0:99999:7:::
对于/bin/cat这个程序文件，root用户运行此程序文件，cat这个进程以root身份访问/etc/shadow, 虽然/etc/shadow文件对root用户的权限是0，但因root是管理员，所以不受限制，仍能访问该文件；当切换至haddoop用户时，haddoop运行/bin/cat此程序文件，cat这个进程以haddoop身份访问/etc/shadow，haddoop不是文件的属主，也不是属组，只能应用other权限，other对/etc/shadow的访问权限为0，所以hadoop不能访问该文件
```bash
[root@centos7 ~]# su - haddoop
Last login: Wed Oct  3 11:59:12 CST 2018 on pts/0
[haddoop@centos7 ~]$ id haddoop
uid=4006(haddoop) gid=4006(haddoop) groups=4006(haddoop)
[haddoop@centos7 ~]$ cat /etc/shadow
cat: /etc/shadow: Permission denied
```
我们通过增加程序文件/bin/cat的SUID权限，实现以haddoop身份/bin/cat此程序文件，访问/etc/shadow；为了避免对/bin/cat原文件的修改，我们将之复制到/tmp目录下，过程如下所示：
```bash
[root@centos7 ~]# cp /bin/cat /tmp/
[root@centos7 ~]# ll /tmp/cat
-rwxr-xr-x. 1 root root 54080 Oct  3 15:18 /tmp/cat
[root@centos7 ~]# /tmp/cat /etc/shadow
root:$6$f6RAFKqJ1Bnk0k0m$v1LyCMQr8J0QBFKB15IhpjWZ6LbnxrorxmEmtxlhFzrcaRQNnwQY0Bxp9qW1cU6pSQYRLDI6lOMLEifh2ozDv0::0:99999:7:::
bin:*:17632:0:99999:7:::
daemon:*:17632:0:99999:7:::
adm:*:17632:0:99999:7:::
lp:*:17632:0:99999:7:::
[root@centos7 ~]# su - haddoop
Last login: Wed Oct  3 15:17:35 CST 2018 on pts/0
[haddoop@centos7 ~]$ /tmp/cat /etc/shadow
/tmp/cat: /etc/shadow: Permission denied
[root@centos7 ~]# chmod u+s /tmp/cat
[root@centos7 ~]# ll /tmp/cat
-rwsr-xr-x. 1 root root 54080 Oct  3 15:18 /tmp/cat
[root@centos7 ~]# su - haddoop
Last login: Wed Oct  3 15:19:46 CST 2018 on pts/0
[haddoop@centos7 ~]$ /tmp/cat /etc/shadow
root:$6$f6RAFKqJ1Bnk0k0m$v1LyCMQr8J0QBFKB15IhpjWZ6LbnxrorxmEmtxlhFzrcaRQNnwQY0Bxp9qW1cU6pSQYRLDI6lOMLEifh2ozDv0::0:99999:7:::
bin:*:17632:0:99999:7:::
daemon:*:17632:0:99999:7:::
adm:*:17632:0:99999:7:::
lp:*:17632:0:99999:7:::
```
管理文件的SUID权限：

   chmod u+|-s FILE

  展示位置：文件属主的执行权限位
     如果属主原本有执行权限，显示为小写s；
     否则，显示为大写S，如下所示
```bash
[root@centos7 ~]# chmod u+s /tmp/cat
[root@centos7 ~]# ll /tmp/cat
-rwsr-xr-x. 1 root root 54080 Oct  3 15:18 /tmp/cat
```
## 特殊权限之SGID

SGID多用于目录文件；当目录文件属组有写权限，且有SGID权限时，那么同时属于此目录属组的用户对该目录有属组的权限，
        且在此目录下新建文件的属组不是该用户的基本组，而是目录文件的属组；
        所以，目录文件属组内的用户在该目录下都有写权限，且可以修改或者删除组内其它用户的文件。
```bash
创建一个目录 /data/tmp/dir
[root@centos7 ~]# mkdir /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir
drwxr-xr-x. 2 root root 6 Oct  3 15:48 /data/tmp/dir

创建一个组testgrp,作为新增用户apple和orange的附加组

[root@centos7 ~]# groupadd testgrp
[root@centos7 ~]# useradd -G testgrp apple
[root@centos7 ~]# id apple
uid=4007(apple) gid=4007(apple) groups=4007(apple),5052(testgrp)
[root@centos7 ~]# useradd -G testgrp orange
[root@centos7 ~]# id orange
uid=4008(orange) gid=4008(orange) groups=4008(orange),5052(testgrp)
[root@centos7 ~]# chmod o+w /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir
drwxr-xrwx. 2 root root 6 Oct  3 15:48 /data/tmp/dir
[root@centos7 ~]# chown :testgrp /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir

以apple身份在目录/data/tmp/dir下创建文件a.apple, 文件a.apple的属主为apple, 属组是apple

[root@centos7 ~]# su - apple
Last login: Wed Oct  3 15:51:05 CST 2018 on pts/0
[apple@centos7 ~]$ cd /data/tmp/dir/
[apple@centos7 dir]$ touch a.apple
[apple@centos7 dir]$ ll a.apple
-rw-rw-r--. 1 apple apple 0 Oct  3 15:53 a.apple

以orange身份在目录/data/tmp/dir下创建文件a.orange, 文件a.orange的属主为orange, 属组是orange

[root@centos7 ~]# su - orange
[orange@centos7 ~]$ cd /data/tmp/dir
[orange@centos7 dir]$ touch a.orange
[orange@centos7 dir]$ ll
total 0
-rw-rw-r--. 1 apple  apple  0 Oct  3 15:53 a.apple
-rw-rw-r--. 1 orange orange 0 Oct  3 15:54 a.orange

为目录/data/tmp/dir新增SGID权限，文件属组执行权限位变为s

[root@centos7 ~]# chmod g+s /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir
drwxrwsrwx. 2 root testgrp 37 Oct  3 15:54 /data/tmp/dir

以apple身份在目录/data/tmp/dir下创建文件b.apple, 文件b.apple的属主为apple, 属组是testgrp

[root@centos7 ~]# su - apple
Last login: Wed Oct  3 15:53:01 CST 2018 on pts/0
[apple@centos7 ~]$ cd /data/tmp/dir
[apple@centos7 dir]$ touch b.apple

以orange身份在目录/data/tmp/dir下创建文件b.orange, 文件b.orange的属主为orange, 属组是testgrp

[root@centos7 ~]# su - orange
Last login: Wed Oct  3 15:54:08 CST 2018 on pts/0
[orange@centos7 dir]$ touch b.orange
[orange@centos7 dir]$ ll
total 0
-rw-rw-r--. 1 apple  apple   0 Oct  3 15:53 a.apple
-rw-rw-r--. 1 orange orange  0 Oct  3 15:54 a.orange
-rw-rw-r--. 1 apple  testgrp 0 Oct  3 16:12 b.apple
-rw-rw-r--. 1 orange testgrp 0 Oct  3 16:13 b.orange
属于目录属组testgrp用户orange对目录应用属组权限rws，可以删除目录下所有文件，可以修改组内其它用户文件
[orange@centos7 dir]$ rm a.apple
rm: remove write-protected regular empty file ‘a.apple’? y
[orange@centos7 dir]$ echo "hello world" >> b.apple
[orange@centos7 dir]$ cat b.apple
hello world
```
管理文件的SGID权限：

 chmod g+|-s FILE

 展示位置：文件属组的执行权限位
    如果属组原本有执行权限，显示为小写s；
     否则，显示为大写S；如下所示：
```bash
[root@centos7 ~]# chmod g+s /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir
drwxrwsrwx. 2 root testgrp 37 Oct  3 15:54 /data/tmp/dir
```

## 特殊权限之Sticky
对于属组或者全局可写的目录，组内所有用户或系统上的所有用户均可在该目录下创建新文件或删除已有文件；若为目录设置sticky位，则用户只能删除属于自己的文件；
```bash

为目录data/tmp/dir新增sticky权限，文件other执行权限位变为t

[root@centos7 ~]# chmod o+t /data/tmp/dir
[root@centos7 ~]# ll -d /data/tmp/dir
drwxrwsrwt. 2 root testgrp 53 Oct  3 16:18 /data/tmp/dir

目录data/tmp/dir新增sticky权限后，组内用户apple不能删除属于orange用户的文件b.orange

[root@centos7 ~]# su - apple
Last login: Wed Oct  3 16:12:21 CST 2018 on pts/0
[apple@centos7 ~]$ cd /data/tmp/dir
[apple@centos7 dir]$ rm b.orange
rm: cannot remove ‘b.orange’: Operation not permitted
[apple@centos7 dir]$ rm b.apple
[apple@centos7 dir]$ ll
total 0
-rw-rw-r--. 1 orange orange  0 Oct  3 15:54 a.orange
-rw-rw-r--. 1 orange testgrp 0 Oct  3 16:13 b.orange
```

## 特殊权限之facl
facl（file access control lists）是文件的额外赋权机制；在文件原有的U,G,O之外，另一层使文件属主能控制赋权给其它用户和组访问文件的权限。

 使用 setfacl命令赋权文件facl

  赋权给用户
    setfacl -m u:USERNAME:MODE FILE

  赋权给组
    setfacl -m g:GROUPNAME:MODE FILE

  撤销赋权
    setfacl -x u:USERNAME:MODE FILE
    setfacl -x g:GROUPNAME:MODE FILE
```bash 
如下示例：
[root@centos7 ~]# useradd pear
[root@centos7 ~]# su - pear
[pear@centos7 ~]$ touch /data/file.pear^C
[pear@centos7 ~]$ echo "i am a pear" >> /data/file.pear
[pear@centos7 ~]$ cat /data/file.pear
i am a pear
[pear@centos7 ~]$ ll /data/file.pear
-rw-rw-r--. 1 pear pear 12 Oct  3 16:50 /data/file.pear
[pear@centos7 ~]$ chmod 660 /data/file.pear
[pear@centos7 ~]$ ll /data/file.pear
-rw-rw----. 1 pear pear 12 Oct  3 16:50 /data/file.pear

apple用户既不是文件/data/file.pear的属主也不是属组，属于other用户，对文件访问权限为0

[root@centos7 ~]# su - apple
Last login: Wed Oct  3 16:30:30 CST 2018 on pts/0
Last failed login: Wed Oct  3 16:51:59 CST 2018 on pts/0
There was 1 failed login attempt since the last successful login.
[apple@centos7 ~]$ cat /data/file.pear
cat: /data/file.pear: Permission denied
[apple@centos7 ~]$ echo "i am an apple" >> /data/file.pear
-bash: /data/file.pear: Permission denied
[root@centos7 ~]# su - pear
Last login: Wed Oct  3 16:49:36 CST 2018 on pts/0
[pear@centos7 ~]$ setfacl -m u:apple:rw /data/file.pear
[pear@centos7 ~]$ ll /data/file.pear
-rw-rw----+ 1 pear pear 12 Oct  3 16:50 /data/file.pear
[root@centos7 ~]# groupadd fruit
[root@centos7 ~]# usermod -aG fruit orange 
[root@centos7 ~]# id orange
uid=4008(orange) gid=4008(orange) groups=4008(orange),5052(testgrp),5053(fruit)
[root@centos7 ~]# su - pear
Last login: Wed Oct  3 16:54:10 CST 2018 on pts/0
[pear@centos7 ~]$ setfacl -m g:fruit:rw /data/file.pear
[pear@centos7 ~]$ ll /data/file.pear
-rw-rw----+ 1 pear pear 12 Oct  3 16:50 /data/file.pear

使用命令 # getfacl FILE 查看文件facl，以如下形式表示：

[pear@centos7 ~]$ getfacl /data/file.pear
getfacl: Removing leading '/' from absolute path names
# file: data/file.pear
# owner: pear
# group: pear
user::rw-
user:apple:rw-
group::rw-
group:fruit:rw-
mask::rw-
other::---
```
文件有facl，访问权限模型发生改变：
 当用户运行某一程序文件时，用户为该进程发起者；进程以用户的身份访问某文件时，文件访问权限模型：
  1. 判断进程的发起者是否为文件的属主，如果是，则应用属主权限
  2. 检查该文件有无针对进程发起者的facl，若有，则应用进程发起者在文件facl中赋予的权限
  3. 否则，检查进程的发起者是否为文件的属组，如果是，则应用属组权限
  4. 否则，检查该文件有无针对进程发起者属组的facl，若有，则应用进程发起者属组在facl中赋予的权限
  5. 否则，应用other权限
如下示例：
```bash
用户apple是文件facl用户，权限为rw-,使用getfacl查看显示如下： user:apple:rw-；所以apple对文件有读写权限

[root@centos7 ~]# su - apple
Last login: Wed Oct  3 16:52:32 CST 2018 on pts/0
[apple@centos7 ~]$ cat /data/file.pear
i am a pear
[apple@centos7 ~]$ echo "i am an apple" >> /data/file.pear
[apple@centos7 ~]$ exit
logout

用户orange属于是文件facl赋权的组fruit，权限为rw-,使用getfacl查看显示如下： group:orange:rw-；所以orange对文件有读写权限

[root@centos7 ~]# su - orange
Last login: Wed Oct  3 16:56:47 CST 2018 on pts/0
[orange@centos7 ~]$ cat /data/file.pear
i am a pear
i am an apple
[orange@centos7 ~]$ echo "i am an orange" >> /data/file.pear
[orange@centos7 ~]$ cat /data/file.pear
i am a pear
i am an apple
i am an orange
```

以上即为本篇所有内容，其中结合实例详述了linux文件特殊访问权限。

  