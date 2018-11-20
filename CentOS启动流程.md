# CentOS系统启动流程
## CentOS5和CentOS6启动流程
![CentOS5和CentOS6启动流程](https://github.com/Summerbluesea/img-folder/blob/master/CentOS6.png)

POST--> BOIS Boot sequence --> MBR Bootloader --> kernel + initramfs(initrd) --> rootfs --> /sbin/init

init程序 -- 系统启动后运行的第一个程序，超级守护进程，用户空间的一切活动均由init管理，需要在系统空间运行通过系统调用实现

init程序类型:
   * SysVinit: CentOS 5使用;各种服务通过脚本实现控制
      配置文件: /etc/inittab
   * Upstart: init, CentOS 6 ；ubantu研发，进程之间基于总线的通信方式D-bus ;兼容SysV特性，仍利用脚本启动服务
      配置文件: /etc/inittab, 所有/etc/init/*.conf文件
   * Systemd: systemd, CentOS 7； systemctl控制各种服务的启动；
        配置文件: /user/lib/systemd/system , /etc/systemd/system 

**CentOS 5和CentOS 6系统服务管理**

### 添加服务
1. 创建服务脚本

Sysv的服务脚本放置于/etc/rc.d/init.d/目录下

```bash
服务脚本示例
#!/bin/bash
# chkconfig: 2345 55 25 
# decription: test scripts
echo "hello srv"
...

2345:定义服务在2,3,4,5运行级别自启动
55：系统启动时服务启动编号，编号大的程序依赖于编号小的服务
25： 系统启动时服务关闭顺序编号
```
2. 使用命令添加
```
 # chkconfig --add SERVICE ; 在每个runlevel对应配置文件创建软连接S*，K*
```
### 服务管理命令

**chkconfig命令使用-更新/查询运行级别对应的system services**
  1. 查看所有服务在各运行级别开机启动状态
    # chkconfig --list [SERVICE_NAME]
  2. 添加服务脚本软连接至各运行级别
    # chkconfig --add SERVICE_NAME 
  3. 删除服务脚本软连接至各运行级别
    # chkconfig --del SERVICE_NAME 
  4. 修改指定运行级别服务脚本软连接
    # chkconfig [--level LEVELS] SERVICE {on|off|reset}
    运行几倍默认为2345
**service命令**
  1. 查看指定服务状态信息
    # service SERVICE_NAME status
  2. 启动指定服务
    # service SERVICE_NAME start 
  3. 关闭指定服务
    # service SERVICE_NAME stop 
  4. 重启/重载指定服务
    # service SERVICE_NAME {restart|reload}
  5. 设定服务开机自启动/关闭开机自启动
    # chkconfig SERVICE_NAME on
    # chkconfig SERVICE_NAME off 

**xinetd管理的服务**
```bash
瞬态服务(Transient service)被xinetd进程管理，例如in.telnet, tftp-server
 配置文件：/etc/xinetd.conf , /etc/xinetd.d/SERVICE 
 服务管理：
 1. 先启动守护进程xinted
    # service xinetd start
 2. 使用chkconfig启动或关闭服务
    # chkconfig SERVICE_NAME {on|off}
```
## CentOS7启动流程
POST--> BOIS Boot sequence --> MBR Bootloader --> kernel + initramfs(initrd) --> rootfs --> /sbin/init(Systemd)

### CentOS 7的systemd

Systemd是linux系统和服务管理器，兼容Sysv和LSB的init服务脚本。
```bash

Systemd新特性：
  1.系统引导时，使用socket和D-Bus激活启动，实现服务并行启动
  2.按需激活进程；所以启动速度块
  3.支持系统状态快照
  4.基于依赖关系定义服务控制逻辑；
  5.通过不同unit功能单元实现系统与服务管理

配置文件进行标识和配置；文件中主要包含了系统服务、监听socket、保存的系统快照以及其它与init相关的信息；
  保存至：
    /usr/lib/systemd/system 
    /run/systemd/system 
    /etc/systemd/system 

Unit的类型：
   Service unit: 文件扩展名为.service ;用于定义系统服务属性
   Target unit:文件扩展名.target;用于模拟实现“运行级别”
   Device unit: 文件扩展名 .device,用于定义内核识别的设备；
   Mount unit: 文件扩展名.mount, 定义文件系统挂载点；
   Socket unit: 文件扩展名.socket, 用于标识进程间通信用的socket文件；
   Snapshot unit: 文件扩展名.snapshot, 管理系统快照；
   Swap unit: 文件扩展名.swap，用于标识swap设别；
   Automount unit:文件扩展名 .automount, 文件系统的自动挂载点
   Path unit: 文件扩展名.path, 用于定义文件系统中的一个文件或目录；
```
### Systemd的服务管理命令

**systemctl COMMAND NAME.service**

注意：systemctl无法管理不是由systemd启动的服务
```bash
1. 查看系统所有服务状态
  # systemctl list-units --type service --all 
2. 查看系统已经激活或尝试激活的服务
  # systemctl list-units --type service
3. 启动指定服务
  # systemctl start name.service
4. 停止指定服务
  # systemctl stop name.service
5. 重启或重载指定服务
  # systemctl reload or restart name.service 
6. 设定指定服务开机自启动/关闭开机自启动
  # systemctl enable name.service
  # systemctl disable name.service
7. 查看指定服务是否可以开机自启动
  # systemctl is-enable name.service 
8. 查看系统所有服务的开机自启动状态
  # systemctl list-unit-files --type service 
9.禁止设定指定服务开机自启动
  # systemctl mask name.service
10 取消禁止设定服务开机自启动
  # systemctl unmask name.service
11. 查看指定服务的依赖关系
  # systemctl list-dependencies name.service 
```

