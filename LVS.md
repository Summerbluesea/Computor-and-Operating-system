# LVS 
负载均衡集群是在站点进行向外扩展(scale out)时，在多台提供相同服务的服务器中间通过负载均衡服务器实现负载均衡的一个集合；
LVS(Linux Virtual Server)是一种LB集群调度器，也称为负载均衡集群服务器，面向客户端请求但本身不提供任何服务，通过其内部定义的调度算法和后端服务器实际承载情况做出决策，将客户端请求转发给后端真正提供服务的某一个服务器，从而实现负载均衡；

构建负载均衡集群要考虑的问题：

    1. 应用程序是否需要会话保持
        有状态连接 stateful：服务器端通过session保存认证客户端身份(cookie)信息及其活动状态于内存中；
        session保持实现方法：
            - session sticky; 用户访问绑定于其第一次访问调度至的后端服务器上，会影响负载均衡；也会有session丢失的可能；
            - session replication; 集群内的后端服务器互相复制session，从而实现session共享，消耗内存资源大，影响性能，规模：3-5台服务器；
            - session server; 设置session存储服务器；

    2. 是否需要共享存储

负载均衡集群(Load Balance Cluster)的实现：
   根据负载均衡服务器在网络通信模型的哪一层实现调度客户端的请求，分类为：
        1. 七层调度器(应用层调度)
            根据应用层协议的报文格式识别客户端身份并根据算法获取数据完成调度；
            需要一个应用层程序，应用层程序通过分析报文格式并根据算法完成调度，再次封装报文转发给后端服务器；需要监听于一个套接字上监听客户端请求；
            最大并发连接数3-4w
        2. 四层调度(传输层)
            根据客户端请求的套接字(ip:port)识别客户端身份并根据算法完成调度；
            在内核完成
            最大并发连接数可达400w 
    
    根据调度器的硬件实现分类为：
        1. 硬负载均衡器(专用硬件)
            F5公司：Big IP
            NetScaler: Citrix
            A10: A10
        2. 软负载(普通的PC-server上通过软件实现负载均衡)
            四层：LVS, Nginx(stream), HAproxy(tcp mod)
            七层：nginx, haproxy, Envoy, Traefik, Kong,...

lvs：Linux Virtual Server 
        VS: Virtual Server；集群中的前端负载均衡器，因其自身不提供任何服务，所以称为虚拟服务器；
        RS: Real Server；RS是后端真正提供服务的服务器；

       

        l4：四层路由器，四层交换机；
           VS：根据请求报文的目标IP和目标协议及端口将其调度转发至某RealServer，根据调度算法来挑选RS；
    
    lvs: ipvs/ipvsadm 
        ipvs:工作于内核空间netfilter的INPUT链的框架；
        ipvsadm: 用户空间的命令行工具，规则管理器，用于管理集群服务及相关的RealServer；

    **lvs的调度算法**
        lvs的调度算法以模块方式内建于linux内核中

        查看内核配置文件中关于ip_vs配置参数：grep -i "ip_vs" /boot/config-kernel_version.x86_64

        调度算法分类：
            静态算法：仅根据算法本身和请求报文特征进行调度；仅关注起点公平；
                rr: round-robin, 轮询；
                wrr: weighted rr, 加权轮询；考虑后端服务器的承载能力，权重大的服务器承载能力大；
                sh: source ip hashing, 可实现session sticky,把来自同一客户端的请求始终调度至同一台RS；[源地址hash值]%[权重之和],计算结果落在哪一个元素编号上就转发给哪一个RS；
                dh: destination ip hashing, 目标地址hash, 典型的应用场景是提供正向代理缓存服务器的负载均衡(将访问的目标地址于代理服务器绑定)
            动态算法：额外考虑后端各RS当前的负载状态进行调度；关注结果公平，确保各RS的负载实时均衡；
                负载的计算方法： 
                    overhead=activeconn*256+inactiveconn 

                ls: least connection; 选择当前连接数最少的RS， 如果所有RS的连接数都相同，则选元素编号小的；
                wls: weighted ls; 考虑后端RS承载能力：overhead=(activeconn*256+inactiveconn)/权重；lvs的默认算法；
                sed：shortest expect delay, 最短期望延迟；overhead=(activeconn+1)/权重，转发给计算结果最小的RS；
                nq: never queue, 永不排队；服务器按权重由大至小排序，请求到达后先给各服务器分配一个，剩下的请求按权重比例分配给各RS；

                lblc: locality based least connection, 用于正向代理缓存服务器的负载均衡，基于dh算法加入动态机制实现的； 
                lblcr: locality based least connection with replication

            不同应用服务的调度算法如何选择：
                短链接，无状态：wrr
                长连接，有状态：wls
                有状态：sh

    **lvs type** 
        lvs的工作拓扑结构和转发机制：

            lvs-nat:多目标DNAT，通过修改请求报文的目标ip和端口为经调度算法挑选出的某后端RS的RIP和RPORT;

            lvs-dr: Direct routing, 通过在原ip报文重新封装帧首部(源mac,目标mac), 目标mac为调度算法挑选出的某后端RS的Rmac 

            lvs-tun: 通过在原ip报文(cip,vip)外再封装一个新ip首部(dip,rip)完成调度；

            lvs-fullnat：通过修改请求报文的源ip(cip -->dip )和目标ip(vip --> rip)完成调度

        lvs-nat:
            1. 拓扑结构
               VS有两个网络接口:1个面向客户端的公网地址(vip), 1个面向RS的私网地址(dip)；
               VS必须是linux系统，RS不限制；

            2. 转发方式特性
               (1) dip和rip要在同一物理网络中，且应该使用私网地址；RS的网关必须指向dip；
               (2) 请求报文和响应报文都要经director转发；director的负荷大易成为瓶颈；
               (3) 支持端口映射，可以修改报文的目标端口；
               
        lvs-dr: 直接路由
            1. 拓扑结构
               VS只需一个网络接口，vip与dip在同一个网络接口，且与RS在同一个二层网络中；
               VS和RS都要配置vip, 解决地址冲突方法：
                    a) 前端路由器的网关与VS的vip的mac地址做静态绑定，确保请求报文路由后发给VS；
                    b) 在RS上使用arptables
                    c) 通过修改RS的内核参数，修改其arp通告和应答级别，以隐藏RS上配置的vip；

                            同一物理网络通信：arp地址解析协议实现，IP --> mac 
                                arp_annouce
                                arp_ignore

            2. 转发特性：
                (1) VS和RS在同一物理网络中，可以是公网地址或私网地址；
                (2) RS的网关必须不能指向dip，以确保响应报文不经VS(director)直接发给客户端；
                (3) 请求报文经director转发，响应报文直接由RS发给客户端；
                (4) 不支持端口映射

        lvs-tun: tunnelling 
            异地容灾
            1. 拓扑结构
                VS和RS都要配置vip地址；
                Rip必须是公网地址，互联网上的可被路由到达的地址；
            2. 转发特性
                (1) vip, dip, rip都应是公网地址
                (2) 请求报文经由director转发，响应报文由RS直接发给客户端；
                (3) director和RS需支持ip隧道

        lvs-fullnat:
            1. 拓扑结构
                VS有两个网络接口:1个面向客户端的公网地址(vip), 1个面向RS的私网地址(dip)；
            2. 转发特性：
                (1) VIP是公网地址，RIP和DIP是私网地址，且通常不在同一IP网络；因此，RIP的网关一般不会指向DIP；
                (3) 请求和响应报文都经由Director；
                (4) 支持端口映射；

                注意：此类型默认不支持；

    ipvsadm管理工具：
        程序包：ipvsadm
        安装： yum -y install ipvsadm 
        配置文件：/etc/sysconfig/ipvsadm-config; 生成的规则可以利用工具保存于此配置文件中，实现永久有效；
        程序文件：
            /usr/sbin/ipvsadm  -->主程序文件
            /usr/sbin/ipvsadm-restore  --> 规则重载工具
            /usr/sbin/ipvsadm-save   --> 规则保存工具
        核心功能：
            集群服务管理：增、删、改；
            集群服务的RS管理：增、删、改；
            查看：

            ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]] [-M netmask] [--pe persistence_engine] [-b sched-flags]
            ipvsadm -D -t|u|f service-address
            ipvsadm -C
            ipvsadm -R
            ipvsadm -S [-n]
            ipvsadm -a|e -t|u|f service-address -r server-address [options]
            ipvsadm -d -t|u|f service-address -r server-address
            ipvsadm -L|l [options]
            ipvsadm -Z [-t|u|f service-address]

        ipvsadm对集群服务和RS分别进行管理：

        管理集群服务：增、改、删；
            增、改：
            ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]]

            删：
            ipvsadm -D -t|u|f service-address

            service-address：
            -t|u|f：
            -t: TCP协议的端口，VIP:TCP_PORT
            -u: UDP协议的端口，VIP:UDP_PORT
            -f：firewall MARK，是一个数字；

            [-s scheduler]：指定集群的调度算法，默认为wlc；

        管理集群上的RS：增、改、删；
            增、改：
            ipvsadm -a|e -t|u|f service-address -r server-address [-g|i|m] [-w weight]

            删：
            ipvsadm -d -t|u|f service-address -r server-address

            server-address：
            rip[:port]

            选项：
            lvs类型：
            -g: gateway, dr类型
            -i: ipip, tun类型
            -m: masquerade, nat类型

            -w weight：权重；

        清空定义的所有内容：
            ipvsadm -C

        查看：
            ipvsadm -L|l [options]
            --numeric, -n：numeric output of addresses and ports 
            --exact：expand numbers (display exact values)

            --connection， -c：output of current IPVS connections
            --stats：output of statistics information
            --rate ：output of rate information

        保存和重载：
            ipvsadm -S = ipvsadm-save

    模拟lvs-nat实验：
        1. 使用docker启动两个httpd容器做RS，宿主机做VS(director)
            # docker run --name web1 -it -v /vols/web1:/var/www/html busybox
            # docker run --name web2 -it -v /vols/web2:/var/www/html busybox
           由于docker启动后默认在FORWARD链生成iptables策略drop所有报文；所以需要修改docker启动参数
            # iptables -P FORWARD ACCEPT

        2. 定义集群服务
            ipvsadm -A -t 172.18.134.134 -s wrr 
        3. 定义RS
            #ipvsadm -a -t 172.18.134.134:80 -r 172.17.0.2:80 -m -w 1
            #ipvsadm -a -t 172.18.134.134:80 -r 172.17.0.3:80 -m -w 2
        4. 测试负载均衡

    lvs-dr
        dr模型中，各主机上均需要配置VIP，解决地址冲突的方式有三种：
            (1) 在前端网关做静态绑定；
            (2) 在各RS使用arptables；
            (3) 在各RS修改内核参数，来限制arp响应和通告的级别；
            限制响应级别：arp_ignore
                0：默认值，表示可使用本地任意接口上配置的任意地址进行响应；
                1: 仅在请求的目标IP配置在本地主机的接收到请求报文接口上时，才给予响应；
            限制通告级别：arp_announce
                0：默认值，把本机上的所有接口的所有信息向每个接口上的网络进行通告；
                1：尽量避免向非直接连接网络进行通告；
                2：必须避免向非本网络通告；

        模拟lvs-dr实验
            (1) 172.18.136.125和172.18.135.51两台主机部署httpd服务做RS；
               
            (2) 172.18.134.134主机是VS(director)
                一个接口配置vip和dip 
            (3) 设定RS内核参数
                arp_announce
                arp_ignore
               /proc/sys/net/ipv4/conf
            (4) 定义集群服务 
                ipvsadm -A -t 172.18.140.1:80 -s wrr

            (5) 定义集群服务
                
                #ipvsadm -a -t 172.18.140.1:80 -r 172.18.136.125:80 -g -w 1
                #ipvsadm -a -t 172.18.140.1:80 -r 172.18.135.51:80 -g -w 2


        FWM：FireWall Mark 
            netfilter：
            target: MARK, This  target  is  used  to set the Netfilter mark value associated with the packet.

            --set-mark value

            借助于防火墙标记来分类报文，而后基于标记定义集群服务；可将多个不同的服务应用使用同一个集群服务进行统一调度；

            打标记方法（在Director主机）：
            # iptables -t mangle -A PREROUTING -d $vip -p $proto --dport $port -j MARK --set-mark NUMBER 

            基于标记定义集群服务：
            # ipvsadm -A -f NUMBER [options]

    lvs持久连接 persistence connections 

    调度器在第一次收到客户端请求并完成调度后将调度结果记录，每一条记录都有固定的TTL，达到TTL设定时间此纪录会自动删除；如果用户在TTL时间范围内再一次发出请求，调度器会查询记录并将请求直接发给相同的RS，实现无论使用任何调度算法，在一段时间内，能够将来自同一个地址的请求始终发往同一个RS；

        firemark+持久连接

            port affinity: 通过防火墙标记将不同应用服务(表现为不同套接字端口)使用一个集群服务统一调度并实现持久服务；

            定义持久集群服务：

            PCC: Persistence Connection per Client，所有端口持久，统一调度
                ipvsadm -A -t 172.18.140.1:0 -s wrr -p [timeout] ; 

            PFWM: 有限几个端口持久，统一调度
                定义防火墙标记：
                # iptables -t mangle -A PREROUTING -d 172.18.140.1 -p tcp -m multiport --dport 80,443 -j MARK --set-mark 7
                定义集群服务：
                # iptables -A -f 7 -s wrr -p 1200 
                定义RS：
                # iptables -a -f 7 -r 172.18.135.51 -m -w 1
            PPC: Persistence Connection per Port; 只对一个端口持久，统一调度
            
            定义RS：
                ipvsadm -a -t 172.18.140.1:0 -r -172.18.136.125 -g -w 1
                ipvsadm -a -t 172.18.140.1:0 -r 172.18.135.51 -g -w 2 


 
