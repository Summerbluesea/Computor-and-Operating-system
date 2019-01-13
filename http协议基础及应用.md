## 网络通信基础
网络中两台主机间通信(资源交换)被抽象为两种模型，OSI七层模型()和TCP/IP四层模型(网络访问层，Internet层，传输层，应用层)，二者的对应关系如下图所示：

其中，传输层(OSI的四层)及以下的层级又称为通信子网，传输层以上又称为资源子网；

通信子网定义了通信细节，实现主机进程与进程之间数据报文传输，在内核空间完成；资源子网实现应用层资源交换，其建立在通信子网之上，在用户空间完成；

    mac地址：标识设备到设备之间通信；物理层+ 数据链路层
    ip地址：标识主机到主机；网络层
    tcp,udp: 标识进程到进程；传输层

tcp和udp都是传输层协议，为应用层的进程提供地址空间，内核用端口号标识进程地址空间；用户空间进程需要与其它进程或其它主机上的进程通信时，进程启动时通过向内核申请端口建立通信通道；

    tcp: Transport Control Protocol，传输控制协议，面向连接的协议，通信前需建立虚拟链路(tcp的三次握手)，有确认、重传和超时机制，通信结束后还需断开链路(tcp的四层挥手)；0-65535个端口

    udp: User Datagram Protocol；无连接协议，通信前无需建立虚拟链路，无确认、重传和超时等特性；传输速度快，数据可靠性不能保证；0-65535个端口；

    0-1023：永久分配给固定的应用使用；特权端口，只有管理员有权使用；22/tcp (ssh), 80/tcp (http), 443/tcp(https)
    1024-41951: 亦为注册端口，但要求并不是特别严格，可分配给程序员注册为某应用使用；11211/tcp,11211/udp (memcached); 3306/tcp(mysql,mariadb)
    41952+:客户端程序随机使用的端口；动态端口，其范围的定义：/proc/sys/net/ipv4/ip_local_port_range

套接字：内核通过套接字来标识一个跨网络通信的进程，也称为进程的文件描述符；
    Socket Domian (根据其所使用的地址):
        AF_INET: Address Family, 基于ipv4地址套接字；ip:port标识一个进程；
        AF_INET6: 基于ipv6地址通信的套接字；
        AF_UNIX: 同一主机上的不同进程之间通信时使用，通过一个UNIX sock文件实现；不经过ip/tcp协议栈；

    当用户空间进程需要建立通信时，会基于套接字的方式调用套接字，从而内核空间申请tcp、udp端口，进程也可以监听与此端口等待其它主机进程的连接请求；

综上所述，通信子网为主机进程间通信建立了通信通道并定义了数据传输细节(tcp/udp)，而进程与进程间资源交换在资源子网实现，由各应用层协议完成；

## http协议

http(HyperText Transfer Protocol)，超文本传输协议，基于html编程语言开发的文本即为超文本，超文本也是一种文本格式。http协议最初开发只是为了传输文本格式数据；

MIME(Multipurpose Internet Mail Extension)机制：多功能互联网邮件扩展，这种扩展可以实现基于文本传输协议将非文本格式数据编码为文本格式进程传输，且在接收方收到数据后可以还原为原有格式；

MIME类型：major/minor 
    text/html -- 超文本文档
    text/plain -- 纯文本文档
    image/jpeg 
    image/gif
    vedio/avi 
    媒体类型的标记决定了客户端浏览器在访问此资源时由哪个web应用程序来进行解析；

在http/1.0以后的版本正式引入MIME机制，使得http协议可以传输非文本数据；尽管http早前就已经加入了MIME机制，但是并不是所有的浏览器都支持所有的媒体类型，这需要浏览器调用其他插件或其他应用程序打开。


http协议版本： 

     http/0.9: 原型版本，不支持多媒体，功能简陋，只支持文本传输；
     http/1.0: 第一个真正广泛使用的版本，支持各种http首部，版本号，http请求的方法，支持MIME机制；缓存机制薄弱；
     http/1.1：增强了缓存功能；
     spdy: 加速http协议的资源获取性能
     http/2.0: 

    http是应用层协议，默认监听于80/tcp，用于完成跨网络文档传输；

一个web页面通常为一个html格式文档，其由多个不同格式的数据资源组成；每一个资源都对应自己的URL，用于标识此资源的访问路径；

URL: Uniform Resource Locator, 统一资源定位，用于标识某服务器上某特定资源的访问路径，任何一个资源在互联网上有一个唯一的标识符；

      URL的组成部分：
        Scheme://Server:Port/path/to/resource 
          URL的resource起始路径映射于server主机上的一个特定目录；资源映射；
            http://www.microsoft.com/images/logo.jpg

一个wen页面的资源分为静态和动态两种：
    静态资源：html文档，图片等，.jpg, .gif, .html, .txt, .js, .css, .mp3,.avi
    动态资源：资源在服务器运行将结果发还至客户端; .php, .jsp

一次完整的http请求和处理过程包括以下七个步骤：

    1. 建立或处理连接
        基于通信子网的tcp协议建立的虚拟链路建立http连接，或关闭某些客户端请求；

        持久连接：tcp连接建立，http建立连接完成一次资源请求后不断开；keepalive；用户在保持连接时长内再次请求无需重新建立连接，可以加速访问资源；但因其占用了一个连接，保持连接时间过长会影响其它用户的访问；在并发访问量大情况下，keepalive的时间要适当缩短；

        非持久连接：tcp连接建立，http建立连接完成一次资源请求后断开；

    2. 接收请求
        接收来自于网络的请求报文对某资源的一次请求过程；

    3. 处理请求
        解析请求报文，确认请求方法和请求的资源等信息；

    4. 获取请求资源

        web服务器：即存放了web资源的服务器，负责向客户端提供其请求的静态资源，或动态资源运行后生成的资源；这些资源存放在服务器主机的本地文件系统某路径下；或经其它路径映射方式定义的location下；

    5. 构建响应报文

    6. 发送响应报文
        对于持久连接：直接发送响应报文给客户端

    7. 记录日志
        做用户行为分析

http协议实现：

    静态web资源服务器：
        apache(httpd)
        nginx
        lighthttpd 

    能够处理动态资源的web服务器：
        IIS
        tomcat, jetty, jboss, resin
        webshpere

## httpd的安装配置和使用

1990s, NCSA（美国计算机安全协会）聚集众多工程师开发了一款能够提供完整服务的web软件，后来项目完成之后，众工程师去往各大IT公司。但是，由于对此项目还是怀有情怀，于是自发发起维护其项目，不断且无偿的为其更新补丁，所以此服务也被称为a patchy server，简称apache，其意为充满补丁的服务器。这诞生了apache最初的雏形；httpd是apache提供web服务的主程序，httpd的版本现已更新至2.4.37；
    CentOS 6 安装光盘提供httpd 2.2.15, 目前apache官方已经宣布httpd-2.2系列end of life；
    CentOS 7 安装光盘默认提供httpd 2.4.6版本；

**httpd的特性：**

   apache用来实现超文本传输协议服务器的主程序，被设计为一个独立运行的守护进程，会建立一个处理进程对每个用户的请求进行处理；

     1. 高度模块化： httpd程序由 core + modules 组成
     2. DSO: (Dynamic Shared Object)，动态模块共享机制，支持模块的动态装卸载；
     3. MPM: Multipath Processing Modules；多路处理模块，响应并发请求；http2.4支持MPM的动态装卸载；
         prefork：多进程模型，每个进程响应一个请求；
             工作模式：
                一个主进程：监听于80/tcp端口，用于建立和处理客户连接请求，管理子进程；子进程的创建和销毁；
                子进程：也称为工作进程，由主进程创建n个子进程，每个子进程处理一个请求；即便没有请求，也会生成多个空闲进程，随时等待请求到达；子进程数量最大不会超过1024个；

         worker：复用的多进程I/O模型，多进程多线程，IIS使用此模型
             工作模式：

                一个主进程：监听于80/tcp端口，用于建立和处理客户连接请求，管理子进程；子进程的创建和销毁；
                子进程：由主进程生成多个子进程，每个子进程负责生成多个线程，每个线程响应一个请求，并发响应请求：m*n 

         event:事件驱动模型
             工作模式：
                一个主进程：监听于80/tcp端口，用于建立和处理客户连接请求，管理子进程；子进程的创建和销毁；
                子进程：由主进程启动多个子进程，每个子进程直接响应n个请求，并发响应请求：m*n；

     4. 丰富的用户认证机制
         basic：基于用户账号和密码进行访问控制；
         digest： 基于消息摘要认证；

     5. 支持第三方模块

**httpd 2.4的配置使用**

    主程序文件：

        /usr/sbin/httpd (默认prefork模型)

        对于http2.2， MPM模型在编译时决定；如果想更换工作模型，需要重新编译；且httpd-2.2对event支持还在测试阶段；rhel提供的rpm包提供编译好的三种模型，想使用哪一个启动对应的程序文件即可；

     配置文件：
        /etc/httpd/conf/httpd.conf 
        /etc/httpd/conf.d/*
        /etc/httpd/conf.modules.d/*


    
    1. 修改监听的Ip和Port；
        Listen [ip:]PORT ; 

    2. 持久连接
        KeepAlive on|off 
        MaxKeepAliveRequests # 最大持久连接个数
        KeepAliveTimeout  #,单位“s”; 持久连接超时时长

    3. MPM配置 
        /etc/httpd/conf.modules.d/00-mpm.conf
         切换使用MPM：uncomment 对应的模块即可

        LoadModule mpm_NAME_module modules/mod_mpm_NAME.so
        重启httpd服务即可
 
        prefork MPM 
        <IfModule prefork.c>
        StartServers       8 ; 初始启动的进程数；
        MinSpareServers    5 ; 最少空闲进程数
        MaxSpareServers   20 ; 最大空闲进程数
        ServerLimit      256 ; 最大启用进程数
        MaxClients       256 ; 每个进程最大并发连接数
        MaxRequestsPerChild  4000; 服务器进程的生命周期内每个子进程的最大响应请求数；达到此数后，主进程会销毁此子进程；
        </IfModule>

        <IfModule worker.c>
        StartServers         4 ; 初始启动的进程数；
        MaxClients         300 ; 每个进程的最大并发连接数
        MinSpareThreads     25 ; 最小空闲线程数
        MaxSpareThreads     75 ; 最大空闲进程
        ThreadsPerChild     25 ; 每个进程生成的线程数
        MaxRequestsPerChild  0 ; 每个进程可以响应的最大请求数；
        </IfModule>

    4. 修改‘Main’ server 的DocumentRoot 

    5. 基于Ip的访问控制法则
         允许所有主机访问： Require all granted 
         拒绝所有主机访问： Require all deny 

         控制特定ip访问：

           Require ip IPADDR:授权指定来源地址的主机访问
           Require not ip IPADDR: 拒绝指定来源地址的主机访问

            IPADDR：
              主机IP
              网络IP：172.18 或 172.18.0.0/16 或172.18.0.0/255.255.0.0 

         控制特定主机(hostname) 访问
            Require Host HOSTNAME 
            Require not host HOSTNAME 

            HOSTNAME : 
               FQDN：
               doamin 
      注意：httpd 2.4的每一个目录都必须授权，默认都不能访问；网页根目录及其各子目录都必须授权；

    
    6. 虚拟主机 
        三种实现方案：
            基于ip: 为每个虚拟主机准备至少一个ip地址
            基于port:为每个虚拟主机准备至少一个专用端口；实践中很少使用
            基于FQDN: 为每个虚拟主机准备至少一个专用FQDN；基于FQDN的不再需要NameVirtualHost指令

         /etc/httpd/conf.d/vhost.conf 
            Listen PORT  ;此处要定义所有Listen的端口；每行定义一个；
            <virtualhost host:port>
                Listen PORT; 如果不是基于端口的定义的虚拟机，此处不需要令行定义；
                ServerName SERVER_NAME
                DocumentRoot 
                Require all granted 
            < virtualhost>
        注意：一般虚拟主机不要与中心主机混用，所以，要使用虚拟主机，先禁用中心主机；
            禁用中心主机：注释DocumentRoot

    7.  http over ssl 

       配置httpd支持https:
           (1) httpd服务器向CA申请数字证书；
              测试：通过私建CA发证书
                (a) 创建私有CA
                (b) 服务器创建证书申请
                (c) CA签署证书
           (2) 配置httpd支持使用ssl，及使用的证书
              查看httpd服务是否支持ssl模块，如果不支持安装
              # yum -y install mod_ssl 

              配置文件：/etc/httpd/conf.d/ssl.conf
              <virtualHost *:443>
              SSLEngine on 
              SSLProtocol all -SSLv2  
                   DocumentRoot 
                   ServerName 
                   SSLCertificateFile 
                   SSLCertificateKeyFile 
            (3) 测试基于https访问响应的主机；
             # openssl s_client [-connet host:port] [-cert filename] [-CApath directory] [-CAfile fielname]

    8. DSO机制
        Dynamic Shared Object: 支持模块动态装卸载

        配置指令实现模块加载:/etc/httpd/conf.modules.d/00-base.conf
        LoadModule <mod_name> <mod_path> 

        mod_path可使用相对路径，相对于ServerRoot(默认为/etc/httpd)指向的路径 
            /etc/httpd/modules --> /usr/lib64/httpd/modules 

        对模块装载或卸载后，service reload httpd ,实现启用或禁用模块；

        # httpd -M  查看所有加载的模块和静态编译的模块

    9. 基于账号访问控制
  
        basic认证： 
                 (1)定义安全域；需要用户认证后方能访问的路径；
                    <directory"">
                       Options None 
                       AllowOveride None 
                       Authtype Basic 
                       AuthName ""
                       AuthUserFile "/PATH/TO/HTTPD_USER_PASSWD_FILE"
                       Require user user1 user2 ..
                     </directory>

                     如果允许账号文件中所有用户访问，则：Require user Valid-user
                 (2)提供账号和密码存储(文本文件存储)
                    默认存放路径：/etc/httpd/conf.d/.htpasswd
                     使用htpasswd命令进行管理
                      htpsaawd [options] passwdfile username 
                         -c:自动创建passwdfile;仅在添加第一个用户时使用；
                         -m: md5加密密码
                         -s: sha1加密
                         -D：删除用户
                  (3) 实现基于组进行认证 
                      <directory"">
                       Options None 
                       AllowOveride None 
                       Authtype Basic 
                       AuthName ""
                       AuthUserFile "/PATH/TO/HTTPD_USER_PASSWD_FILE"
                       AuthGroupfile "PATH/TO/HTTPD_GROUP_PASSWD_FILE"
                       Require group GROUP_NAME 
                     </directory>

                  要提供组文件和组内用户文件 /etc/httpd/conf.d/.htgroup 
                     GROUP_NAME: user1 user2 

    10. 路径别名
        DocumentRoot: The directory out of which you will serve your documents. By default, all requests are taken from this directory, but symbolic links and aliases may be used to point to other locations；

        定义路径别名，将用户对网页文件根目录下的某个子目录映射至server的文件系统上的某个目录
        mkdir /var/www/html/bbs 
        mkdir /forum 
    
        /etc/httpd/conf/httpd.conf

        Alias /bbs "/forum"

        <Directory "/forum">
        AllowOverride None
        Require all granted
        </Directory>