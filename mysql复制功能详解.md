# MySQL复制详解

mysql的复制架构是MySQL向外扩展的解决方案，从节点向主节点请求二进制日志中的事件于本地并执行replay从而完成与主节点数据同步；通过复制实现每个节点有相同的数据集；本文将详细介绍MySQL服务器的主从架构复制、主主架构复制、半同步复制、复制过滤器以及加密复制的实现；

复制的功用：

    数据分布：数据集在主从节点都有存放；
    负载均衡：主要针对读操作而言；
    数据备份： 冗余，辅助完成备份;
    高可用和故障切换：
    MySQL升级测试：

一. mysql主从复制

主从复制工作过程：

      从节点启用两个线程：
          I/O thread: 从MASTER请求二进制日志，并保存于本地的中继日志中；
          SQL thread: 从中继日志中读取日志事件，在本地完成重放；

      主节点：
          dump thread:为每个slave的I/O thread 启动一个dump线程，用于向其发送binary log events

主从复制特点：

    1. 异步复制
        主节点发生写入操作时要不要等待从节点复制完成并反馈写入结束再告诉用户写入完成；
        异步复制即主节点本地写入完成即告诉用户完成，不等待从节点replay完成并反馈；
 
    2. 主从数据不一致比较常见；

主从复制实现：

    主节点：
    
        (1) 启用二进制日志
            [mysqld]
            log_bin=mysql-bin
        (2) 为当前节点设置一个全局唯一的ID号
            [mysqld]
            server_id=#
        (3) 创建有复制权限的用户账号
            REPLICATION SLAVE, REPLICATION CLIENT 

    从节点：

        (1) 启用中继日志
            [mysqld]
            relay_log=relay-log 
            relay_log_index=relay-log.index
        (2) 为当前节点设置一个全局唯一的ID号
        (3) 使用有复制权限的用户账号连接至主节点并启动复制线程
             mysql> CHANGE MASTER TO MASTER_HOST='', MASTER_USER='', MASTER_PASSWORD='', MASTER_LOG_FILE='', MASTER_LOG_POS=
             查看帮助：
             mysql> help CHANGE MASTER TO 
             查看从节点状态
             mysql> SHOW SLAVE STATUS;

             启动从节点同步相关线程：
             mysql> START SLAVE [IO_THREAD|SQL_THREAD];

    注意：如果主节点已经运行一段时间，需先dump一份完整数据给从节点，并记录dump时间节点二进制日志文件和位置；从节点先导入完整dump数据，再执行CHANGE MASTER TO操作；

        # mysqldump --all-databases --master-data=2   --flush-logs > /tmp/allbackup.sql
        # scp /tmp/allbackup.sql slave_host_ip:/root/
        # mysql < allbackup.sql (slave执行)

复制架构要注意的问题：

      1. 限制从节点为只读

      read_only=1; 此限制对拥有super权限的用户均无效；

      阻止所有用户：从节点开一个终端执行全局读锁命令保持不退出
      mysql> FLUSH TABLES WITH READ LOCK;

      2. 如何保证主从复制的事务安全？

      在master节点启用参数：
        sync_binlog= 1 ;
        sync_master_info=ON; 
      当事务提交时，必须将主节点binlog缓冲区事件写入磁盘上的二进制日志中；但会增加主节点I/O压力

      如果用到的为innodb存储引擎：
        innodb_flush_logs_at_trx_commit=ON ; 事务提交时立即将事务日志中的event刷写至磁盘的二进制日志中；
        innodb_support_xa=ON ; 指定innodb支持分布式事务

      在slave节点启用参数：
        skip_slave_start=ON；
        手动启动从节点同步线程；

        sync_relay_log=ON ; 从节点同步relay_log

        sync_relay_log_info=ON; 从节点同步relay_log_info, SQL thread

二. 主主复制模型

两个节点互为主从，都可以读写数据：

      (1) 数据可能会不一致；因此，慎用此模型；
      (2) 自动增长id的字段
          配置一个节点使用奇数id
            auto_increment_offset=1
            auto_increment_increment=2 
          另一个节点使用偶数Id 
            auto_increment_offset=2 
            auto_increment_increment=2 

      配置步骤：

      (1) 各节点使用一个唯一的server_id;
      (2) 都启用binlog和relay log; 
      (3) 创建拥有复制权限的用户；
      (4) 定义自动增长id字段的数值范围分别为奇偶；
      (5) 均把对方指定为主节点，并启动复制线程；

三. 半同步复制

  同步复制、异步复制、半同步复制：主节点发生写入操作时要不要等待从节点复制完成并反馈写入结束再告诉用户写入完成；

    异步复制：主节点本地写入完成即告诉用户完成，不等待从节点replay完成并反馈；
 
    同步复制：主节点等待所有从节点写入完成后反馈用户写入完成；

    半同步复制：主节点只等待某几个从节点(半同步复制节点)反馈写入完成即告诉用户写入完成，不必等待所有从节点写入完成；既确保了数据安全性，加快了写入操作；

半同步复制实现：

    (1) 先配置主从节点，实现主从复制
    (2) 主节点安装半同步复制插件
        mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
        mysql> SHOW PLUGINS; 
        mysql> SHOW GLOBAL VARIABLES LIKE '%semi%';

    (3) 主节点配置
        开启主节点半同步参数
        mysql> SET GLOBAL  rpl_semi_sync_master_enabled = ON ; 
        mysql> rpl_semi_sync_master_timeout = 10000; 主节点等待从节点半同步时长

    (4) 从节点安装半同步插件
        mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
        mysql> SHOW PLUGINS;
        mysql> SHOW GLOBAL VARIABLES LIKE '%semi%';

    (5) 从节点配置
        开启从节点半同步复制参数
        mysql> SET GLOBAL  rpl_semi_sync_slave_enabled = ON 

四. 复制过滤器

  让从节点复制指定的数据库，或指定的数据库的指定表；

    有两种实现方式：

    (1) 主服务器仅仅向二进制日志中记录与特定数据库(特定表)相关的事件；
        问题：利用二进制日志实现时间点还原无法实现，不建议使用；

        binlog_do_db=         #数据库白名单
        binlog_ignore_db=      # 数据库黑名单列表，使用逗号分隔

    (2) 从服务器SQL_THREAD 在replay中继日志中的事件时，仅读取指定数据库(特定表)相关的事件应用于本地；
        问题：会造成网络及磁盘IO浪费；

        replicate_do_db=        # 复制白名单
        replicate_ignore_db=    # 复制黑名单

        还可以针对特定的tables实现过滤，可能会设置的参数：

        replicate_do_table=
        replicate_ignore_table=
        replicate_wild_do_table=
        replicate_wild_ignore_table= 


五. 基于ssl复制

      mysql> help CHANGE MASTER TO ; 查看SSL复制相关参数
      安全的主从连接

      (1) 确认主从服务器是否支持openssl

          mysql> SHOW VARIABLES LIKE '%ssl%';
          have_openssl=YES 
          have_ssl=YES 

      (2) 准备CA
          # (umask 066;openssl genrsa -out 2048 > cakey.pem)
          openssl req -new -x509 -key cakey.pem -out cacert.pem -days 3650

      (3) master配置证书和私钥。并且创建一个要求必须使用SSL连接的复制用户账号

          #  openssl req -newkey rsa:2048 -days 365 -nodes -keyout master.key > master.csr
          #  openssl x509 -req -in master.csr  -CA cacert.pem -CAkey cakey.pem -set_serial 01 > master.crt
          #  openssl x509 -in master.crt -noout -text  --> 查看证书内容

          mysql> GRANT ALL ON *.* TO 'ssluser'@'%' IDENTIFIED BY 'sslpass' require ssl;

      主节点配置文件： /etc/my.cnf 
          [mysqld]
          ssl
          ssl-ca=/etc/my.cnf.d/ssl/cacert.pem
          ssl-cert=/etc/my.cnf.d/ssl/master.crt
          ssl-key=/etc/my.cnf.d/ssl/master.key

      (3) slave 配置证书和私钥。使用CHANGE MASTER TO命令时指明SSL相关选项；

          # openssl req -newkey rsa:2048 -days 365 -nodes -keyout slave.key > slave.csr
          # openssl x509 -req -in slave.csr  -CA cacert.pem -CAkey cakey.pem -set_serial 02 > slave.crt

          mysql> CHANGE MASTER TO 
          MASTER_HOST='172.18.135.51',
          MASTER_USER='rpluser',
          MASTER_PASSWORD='rplpass',
          MASTER_SSL=1,
          MASTER_SSL_CA='/etc/my.cnf.d/ssl/cacert.pem',
          MASTER_SSL_CERT='/etc/my.cnf.d/ssl/slave.crt',
          MASTER_SSL_KEY='/etc/my.cnf.d/ssl/slave.key',
          MASTER_LOG_FILE='master-bin.000022',
          MASTER_LOG_POS=926

      或者：如果主服务器可以重启，在客户端的配置区配置
          [mysql]
          ssl-ca=/path/to/cacert.pem 
          ssl-cert=/path/to/slave.crt 
          ssl-key=/path/to/slave.key 

          然后，在从节点执行CHANGE MASTER TO 
          mysql> CHANGE MASTER TO MASTER_HOST='',MASTER_USER='repluser',MASTER_PASSWORD='replpass',MASTER_SSL=1,MASTER_LOG_FILE='',MASTER_LOG_POS=

