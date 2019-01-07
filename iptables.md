# linux防火墙
防火墙功能在于隔离网络或主机，对进出网络或主机的数据包基于一定的规则检查，并定义了被规则匹配的数据包处理行为，从而实现隔离功能；linux在内核定义了netfilter框架，netfilter框架在报文流入和流出的路径上定义了5个hooks，分别为：
  
  报文流入路径：
   - prerouting：到达本机网卡路由之前 
      - input：数据包目标地址就是本机，路由后送往应用程序前
      - forward：数据包目标地址不是本机，路由后送往转发前
  
 报文流出路径：
   - output: 数据包由本机的某个应用程序发出
   - postrouting: 数据包出站经路由后

 iptables程序将netfilter定义的5个hook映射为5个链，分别为PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING，用户可以通过iptables工具在这5个链上定义匹配规则和target，以实现不同功能。iptables支持自定义链，自定义链做为内置链上规则的target被引用才能生效；

 iptables功能：
   1. filter:数据包过滤，可以在INPUT, FROWARD, OUTPUT 链上设置；
   2. nat: 地址转换，在PREROUTING, POSTROUTING链上设置，INPUT和OUTPUT链上特殊情况使用  
   3. mangle 
   4. raw 

## iptables工具使用
**安装** 
 yum install iptables-services 

**iptables filter功能实现** 
```bash
iptables [-t filter] COMMAND chain rule-specification 

 所有链的默认策略是所有数据包ACCEPT 

 COMMAND: 
  管理规则：-A(append), -I(insert), -D(delete), -R(replace) , -F(flush) 
  管理链：
     -N：添加新的自定义链
     -X：删除一个空且引用计数为零的自定义链 
     -P(policy)：设置指定链的默认策略
     -E(rename)
     -Z：置零指定链或指定链指定规则的计数器
  查看定义的规则： -L ，组合使用vnl, --line-numbers(显示规则号) -x(显示精准数据 )

主机防火墙在INPUT和OUTPUT链实现


 rule-specification : 
  匹配条件+target组成:[-m matchname] [match-options] -j target 

  匹配条件： 
   通用匹配：在网络层实现，可省略写-m matchname  
       [!] -p, --protocol protocol
       [!] -s, --source address[/mask]
       [!] -d, --destination address[/mask]
       [!] -i, --in-interface 
       [!] -o, --out-interface 

   扩展匹配：必须指明 -m matchname 
      隐式扩展：
       (1) -p tcp:
               --sport 
               --dport 
               --tcp-flags mask comp 
               mask:指定要检查的标志位
               comp:指定mask中规定的标志位必须为1的标志位，comp没有指明的则必须为0 
       (2) -p udp: 
              --sport 
              --dport 
       (3) -p icmp:
              --icmp-type {name|code}
                echo-request 8 
                echo-reply 0
      
      显示扩展
       (1) multiport 
           --sports 21:22,80,139 ; 可以给区间21:25,表示21到25连续端口
           --dports 
       (2) iprange 
           --src-range from [-to]:
           --dst-range from [-to]:
         # iptables -I INPUT 4 -p icmp --icmptype 8 --src-range 172.18.0.60-172.18.0.70 -j ACCEPT
       (3) set ；通过ipset工具定义ip集,实现同时控制此ip集中的主机
         依赖于实现定义好的ipset：
         安装ipset: yum install ipset 
         创建ip集： ipset create IPSET_NAME TYPE
            TYPE: hash:net, hash:ip 
         向定义的ip集中加入ip: ipset add IPSET_NAME ip 

       (4) string 
          对报文中的应用层数据做字符串匹配检测；

         --string PATTERN --algo {bm|kmp}
         
       (5) . time 
            根据报文到达时间与指定的时间范围匹配检测；
               --datestart：2007-12-24
               --datestop：
               --timestart：12:30:[ss]
               --timestop 
               --weekdays：1-7
               --monthdays: 1-31 
        (6) . connlimit 
         每个客户端ip做并发连接数数量限制匹配;
            --connlimit-upto #: 
            --connlimit-above #: 

            通常与允许和拒绝规则配合使用

            # iptables -I INPUT 2 -p tcp --dport 22 -m connlimit --conlimit-above 2 -j DROP
         
         (7). limit 
            客户端ip的报文传输速率限制；
            --limit # {/second|min|hours|}
            --limit-burst # 
         (8). state 
            根据“conntrack”检查匹配连接状态；基于连接追踪机制
              --state STATE1,STATE2
            连接状态 
                  NEW 
                  ESTABLISHED 
                  RELATED 
                  UNTRACKED : 未追踪的连接
                  INVALID : 
                  SNAT:源地址转换
                  DNAT:
            连接追踪表：存储于内核的内存中
               能追踪的最大连接数设定:/proc/sys/net/nf_contrack_max;
                  可以自定义， 配置文件 /etc/sysctl/sys.conf 
                                    /etc/sysctl.d/*.conf 
               修改后让sysctl重读配置文件即生效：
               # sysctl -p /etc/sysctl.d/*.conf 

         其中RELATED模块需手动加载对应的模块nf_conntrack_*： 
            # modprobe nf_conntrack_*
   处理动作：-j target 
        ACCEPT 
        DROP
        REJECT 
        自定义链

iptables工具生成的规则即时生效，但不会永久有效，永久有效需写入配置文件:/etc/sysconfig/iptables 
  保存工具：iptables-save > /path/to/file 
  重读文件中保存的规则： iptables-restore < /path/to/file 

iptables 实现网络防火墙
 FORWAR链实现，控制请求报文
 1. 受控方是请求发出方
   示例：只允许172.18.134.142主机外网其它主机，不允许外网其它主机ping172.18.134.142 ；
   实验环境：172.18.134.134主机开启核心转发，做router使用
   # iptables -A FORWARD -m state --state ESTABLISHED -j ACCEPT
   #  iptables -I FORWARD 2 -s 172.18.134.142/16 -p icmp --icmp-type 8 -m state --state NEW -j ACCEPT 
   # iptables -t filter -A FORWARD -j DROP
   # iptables -vnL
   5   420 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state ESTABLISHED
   1    84 ACCEPT     icmp --  *      *       172.18.0.0/16        0.0.0.0/0            icmptype 8 state NEW
   0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0           

 2. 受控方是请求响应方
   示例：172.18.134.142主机的mysql服务只能本地网络访问
   # iptables -R FORWARD 2 -s 172.18.0.0/16 -d 172.18.134.142 -p tcp --dport 3306 -m state --state NEW -j ACCEPT
   # iptables -A FORWARD -m state --state ESTABLISHED -j ACCEPT
   # iptables -t filter -A FORWARD -j DROP



```
**iptables nat功能实现** 
```bash
1. SNAT 静态源地址转换
   请求报文做SNAT，NAT服务的连接追踪机制会自动为响应报文做DNAT，无需配置；在postrouting链设置SNAT规则；
   命令格式： iptables -t nat COMMAND chain -j SNAT --to-source ipaddr 
 示例：
 实验环境：192.168.48.100和192.168.48.103是本地网络的两台主机，172.8.134.134是网关服务器；本地主机通过做SNAT转换隐藏且做到可以与互联网通信；
 MASQUERADE：地址伪装；只在POSTROUTING 的target 
#iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -d 172.18.0.0/16 -j SNAT --to-source 192.168.48.77
# iptables -t nat -A POSTROUTING -s 192.168.88.0/24 ! -d 192.168.88.0/24 -j MASQUERADE 
2. DNAT 目标地址转换
   请求报文做DNAT，NAT服务基于连接追踪机制会自动为响应报文做SNAT，无需配置；在PREROUTING 链设置DNAT规则；只针对有限的端口进行；
 iptables -t nat COMMAND chain -j DNAT --to-destination ipaddr:port 
 #  
  
3. Full NAT
   请求报文的目标地址和源地址都做地址转换；

4. PORTMAP 端口映射
  --to-port PORT 
  一个公网ip地址,在DNAT做端口映射
  不同目标端口转给不同服务，以达到DNAT上一个ip公网ip提供多种服务；
  配置多个公网地址，做DNAT，以隐藏地址；
      
         


 

      



 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 

  