# DNS和BIND内容详解
互联网中的每一台主机都有一个或多个对应的ip地址，为了用户更方便的访问互联网内的主机，而不用记ip地址数串，DNS(Domain Name System)域名解析系统提供了一个域名和ip地址映射的分布式数据库，用户通过域名访问互联网中的某一主机，由DNS服务器提供域名解析服务，将该域名对应的ip地址返回用户从而建立连接；域名是互联网上的身份标识，是不可重复的唯一的标识资源；

DNS是应用层协议，工作于资源子网，C/S架构，服务器端监听于53/udp和53/tcp；
  
BIND: Berkley Internet Name Domain, ISC发行和维护(www.isc.org); BIND是DNS协议的开源实现，提供dns服务；
## DNS基础
```bash
1. DNS域名空间

  DNS将域名空间分层管理，域名空间呈倒置的树状结构，由上至下依次为根域、一级域、二级域、三级域...
  - 根域 .  
    - 一级域 Top Level Domain, tld 
      com. net. edu. gov. org. mil. ...
      - 二级域  例如：baidu.com. 
        - 三级域  一般在三级域内即是各主机了，如果需要还可以继续授权子域

2.名称服务器；Name Service

  每个域内负责解析本域内主机域名的主机称为名称服务器，全球共有13个根域服务器;

3. DNS查询类型

  递归查询：发送一次请求就可以得到最终结果
  迭代查询：发送查询请求得到的结果可能是参考性答案，需发送多次请求得到最终答案

4. DNS解析类型

  FQDN -- Full Qualified Domian Name，全限定域名，例如 www.baidu.com. 
  正向解析：FQDN --> IP 
  反向解析：IP --> FQDN
  DNS正向解析和反向解析在不同的名称空间进行，需要两个不同的名称解析库；

5. DNS服务器类型

  主DNS服务器：维护所负责解析的域内解析库的服务器；解析库内记录了本域内所有主机域名与ip地址的映射关系；解析库由管理员负责维护；
  从DNS服务器：从DNS服务器定义与主DNS服务相同的解析域，但无需自己配置解析库，从主DNS服务器或其它从DNS服务器“同步”解析库；
  只缓存DNS服务器：自身不负责任何域，它接受客户端的递归查询请求，客户端的域名解析请求由它迭代查询缓存至本地再反馈至客户端主机；
  转发DNS服务器：定义DNS的转发功能，如果不是自己域内解析请求，将请求转发给指定的DNS服务器；需要指定的DNS服务器能为解析请求做递归查询；

6. 区域解析库

  区域解析库表现为一个文件，文件内每一条记录称为资源记录RR(Resource Record); 主要的RR类型有SOA, NS, MX, A, AAAA,CNAME， PTR；
  资源记录定义的格式：
       通用语法：name [TTL] IN  rr_type   value 
         TTL:客户端得到结果的可缓存时长
         注意：(1) TTL可从全局继承；在解析库头部定义$TTL
              (2) @可用于引用当前区域的名称; $origin 
  资源记录的类型：
  SOA: Start Of Authority; 每个解析库必须有且仅有一条SOA记录，且必须是第一个RR；
   示例：moubao.com.  86400 IN SOA ns.moubao.com. nsadmin.moubao.com. (
                     2018112701; serial
                     2H        ; refresh time
                     10M       ; retry
                     1W        ; exprie
                     1D        ; 否定答案的TTL
        )
  NS: Name Service; 记录本域内的名称服务器是的名称，每个NS记录的名称服务器后续都有一个对应的A记录；
   示例：  moubao.com. IN  NS  ns.moubao.com. 
  MX: Mail EXchange; 记录本域内的邮箱服务器的名称；每个MX记录的名称服务器后续都有一个对应的A记录；
   示例：  moubao.com. IN  MX  mx.moubao.com.
  A：记录域名与ipv4地址的对应关系；
   示例：  www.moubao.com.  IN  A  1.1.1.1 
          www.moubao.com.  IN  A  1.1.1.2
          web.moubao.com.  IN  A  1.1.1.1
    一个主机的FQDN可以有多条记录定义不同的值，此时DNS服务器以轮巡方式响应;
    一个值也可能有多个不同的定义名字；通过多个不同的名字指向同一个值进行定义；此仅表示通过不同的名字可以找到同一个主机
    通过A记录实现泛域名解析：
            *.moubao.com.  IN   A   1.1.1.4 
            moubao.com.  IN   A   1.1.1.4
  AAAA：记录域名与ipv6地址的对应关系；
  PTR: PoinTeR; 反向解析记录；反向解析库中记录
            4    IN  PTR   www.moubao.com 或
            4.1.1.1-in-addr-arpa.   IN  PTR  www.moubao.com 
  CNAME：别名激励
       web.moubao.com   IN    CNAME   www.moubao.com
7. 域名注册

   域名注册：向代理商申请
   代理商：万网，新网；godaddy

每台主机在本地有一个解析文件/etc/hosts，每行记录一条FQDN--ip映射关系，当我们的主机访问互联网一个域名时，执行的一次完整的域名解析过程如下：

Client --> 查找本机/etc/hosts文件中是否有匹配记录 --无-->请求DNS Service(向指定的DNS服务器请求DNS服务) --> 查找local DNS cache --无--> DNS Server --> DNS recursion --无--> DNS server cache --无--> DNS server iteration(迭代查询) --> results --> Client 
```
## BIND使用
**BIND安装配置**
```bash
程序包名bind, 服务名named;
需要安装的程序包：bind, bind-utils, bind-libs
  bind-utils:提供DNS解析测试工具； dig, host, nslookup, nsupdate

1. 安装bind 
    yum install bind 
2. 配置
    主配置文件：/etc/named.conf 
       acl定义：acl acl_name{}；把一个或多个主机归并为一个集合，并通过一个统一的名称调用；       
       全局配置定义：options {}
       日志子系统配置：logging {}
       区域定义：本机能为哪些zone进行解析，就要定义哪些域
         zone "ZONE NAME" IN {}
           必需提供：根区域，本地回环地址，localhost
    主配置文件：/etc/named.rfc1912.zones
       定义本机的解析域；
    辅助配置文件：/etc/rndc.key 
    解析库文件：/var/named/ZONE_NAME.zone
      注意：(1)一台物理服务器可同时为多个区域提供解析服务
            (2)必须要有根区域解析库; named.ca 
            (3) 应该有两个(甚至更多，包括ipv6)实现localhost和本地回环地址的解析库；
3. 启动服务
   默认只监听在127.0.0.1地址下；所以不能被除本机外的其它主机访问；

4. 配置缓存DNS服务器
  将配置文件/etc/named.conf中的监听地址添加本机可以访问互联网的ip地址；
  设定可以提供递归查询：recursion yes 即可；
  可以定义允许查询的主机 allow-query
```
**DNS主从服务器实现**
```bash

1. 配置主DNS服务器

 在实现缓存服务器的基础上
 (1)在/etc/named.rfc1912.conf文件中定义解析域, type为master；
     zone "ZONE_NAME" IN {
        type master;
        file "ZONE_NAME.zone";
        allow-query { any; };
        allow-transfer { }
        allow-update { none; };
    };
 (2) 定义区域解析库
   /var/named/ZONE_NAME.zone
    内容：
     宏定义：$TTL 1D
            [$ORIGIN moubao.com.]
     资源记录：
 (3) 语法检查命令
    主配置文件 --> named-checkconfig
    解析库 --> named-checkzone  zonename /path/to/zone_file
 (4) 注意自己定义的解析库文件属性
    chmod 640 moubao.com.zone
    chown :named moubao.com.zone 
 (5) named服务重新加载配置文件
     rndc reload 

2. 配置从DNS服务器

  同样在实现缓存服务器的基础上
 (1)在/etc/named.frc1912.conf文件中定义与主DNS相同的解析域, type为slaves；
     zone "ZONE_NAME" IN {
        type slave;
        masters { MASTER_IP; };
        file "slaves/ZONE_NAME.zone";
        };
 (2)从DNS服务器重新加载配置文件
    执行 rndc reload，从DNS服务器自动同步主DNS服务器解析库文件至/var/named/slaves/目录下
    在主DNS服务器的区域解析库文件中定义了主从服务协调属性以及否定答案的统一的TTL
    @	IN SOA	@ rname.invalid. (
					0	; serial，解析库的序列号，当解析库记录发生变化，序列号递增，触发主从服务器解析库同步
					1D	; refresh，从服务器多长时间从主服务器同步一次数据；
					1H	; retry，从服务器的同步请求没有得到主服务器回应时，多长时间再试
					1W	; expire，从服务器的同步请求一直没有得到回复，多长时间关闭dns解析服务；
                    3H ); minimum， 否定答案的缓存时长；
主从DNS服务器实现：
 1. 从服务器应该为一台独立的名称服务器
 2. 主服务器的区域解析库文件中必须有一条NS记录指向从服务器；
 3. 从服务器只需要定义区域，而无需提供解析库文件；解析库文件应该放置于/var/named/slaves/目录中；
 4. 主服务器得允许从服务器作区域传送；默认即是允许的
 5. 主从服务器时间应同步，可通过ntp进行时间同步；
 6. bind程序的版本应该保持一致；否则，应该从高，主低；

3. 反向解析域主从DNS服务器配置

 <1> 反向解析域主DNS服务器配置

    在实现缓存服务器的基础上
    (1)在/etc/named.rfc1912.conf文件中定义解析域, type为master；
        反向区域：
    区域名称：网络地址反写.in-addr.arpa.
        172.18.0.0. --> 18.172.in-addr-arpa.
        zone "ZONE_NAME" IN {
        type master;
        file "网络地址.zone";
        };
    (2) 定义解析库文件
        注意：不需要MX和A，AAAA记录，以PTR记录为主；
            反向解析库依赖于正向解析库中的NS解析服务器的ip地址
    示例：
            $TTL 1D
            $ORIGIN 18.172.in-addr.arpa. 
            @	IN	SOA	moubao.com. admin.moubao.com. (
                            2018112701
                            1H
                            10M
                            1W
                            1D )
                    NS	ns1.moubao.com. 
                    NS	ns2.moubao.com.
            134.134	IN	PTR	www.moubao.com.
            51.135	IN	PTR	ns2.moubao.com.
            1.134	IN	 PTR	mx1.moubao.com

 <2> 反向解析域主DNS服务器配置

    同样在实现缓存服务器的基础上
    (1)在/etc/named.frc1912.conf文件中定义与主DNS相同的解析域, type为slaves；
        zone "ZONE_NAME" IN {
            type slave;
            masters { MASTER_IP; };
            file "slaves/ZONE_NAME.zone";
            };
    示例：
       zone "18.172.in-addr.arpa." IN {
        type slave;
        masters { 172.18.134.134; };
        file "slaves/172.18.zone";
       };

    (2)从DNS服务器重新加载配置文件
      rndc reload
以上即为正向解析域与反向解析域主从DNS的配置；
```
**子域授权**
```bash

实现子域授权，需要在父域主名称解析服务器的解析库中定义指向子域的NS记录；
正向解析区域的子域实现方法：
示例：
 父域的解析库文件中定义一个子域, moubao.com.要授权名称为ops的子域，在moubao.com.域解析库中添加如下RR：
        ops.moubao.com. IN NS ns1.ops.moubao.com.
        ops.moubao.com. IN NS ns2.ops.moubao.com.
        ns1.ops.moubao.com. IN A 1.1.1.1
        ns2.ops.moubao.com. IN A 1.1.1.2 
```
**转发服务器**
```bash

定义转发服务器：
  不是当前服务器负责的解析域都转发给指定的DNS服务器，指定的服务器可以是根域服务器或某一个确定的DNS服务器
  注意：被转发的服务器需要能够为请求着做递归查询，否则，转发请求不予进行；
   (1) 全部转发：凡是非本服务器所负责解析区域的请求，统统转发给指定的服务器；
       在/etc/named.conf的option中定义
       option {
         forward {first|only}
         forwarders { dns_server_ip; }
       }
   (2) 区域转发：只转发特定区域的请求至指定的服务器；可以在子域服务器中定义父域解析请求转发，把父域的解析请求都转发给父域名称服务器；
      /etc/named.rfc1912.conf 
       zone "ZONE_NAME" IN {
         type forward;
         forward {first|only};
         forwarders { dns_server_ip; };
       };
    注意：实现转发需关闭dnssec功能；
         dnssec-enable no;
        dnssec-validation no;
    如果区域转发和全局转发均开启，则能够被区域转发匹配的解析请求将转发给区域转发；不能匹配的转发给全局转发
    first:先转发给转发器，如果转发器无响应，则自己做迭代查询
    only:仅转发给转发器
```
**BIND view**
```bash

bind通过定义不同的view实现将客户端的访问分组，通过不同的解析库文件实现对同一个域内主机的解析，这样用户访问的是同一个域名，但得到的ip地址不同；
   一个bind服务器可定义多个view；
   每个view中可定义一个或多个zone；
   每个view用来匹配一组客户端；
   每个view内或多个view内可能需要对同一个域进行解析，但使用的区域解析库文件不同；
 注意：
  (1) 一旦启用view，所有该view的zone都必须定义在view中；
  (2) 仅有必要在匹配到允许递归请求的客户所在view中定义根域；
  (3) 客户端请求到达时，服务器自上而下检查每个view服务的客户端列表进行匹配；

在域配置文件中定义view：/etc/named.rfc1912.conf 

定义每个view和其负责的客户端组(match-clients)；
 match-clients可以通过在/etc/named.conf 定义acl实现；
通过不同的解析库文件为相同的zone解析，使不同客户端访问相同主机的DNS解析得到的ip地址不同

       view VIEW_NAME {
          match-clients {};
          zone "ZONE_NAME" IN {
            type 
            filename ""
          };
       };
    大部分可以用在option的指令都可以在view中定义；
```
**BIND中基础的安全相关配置**
```bash

 1.acl:把一个或多个主机归并为一个集合，并通过一个统一的名称调用；
      acl acl_name {
        ip1;
        ip2;
        net1/prelen;
      };
  BIND有4个内置的acl:
      none:没有一个主机；
      any:任意主机；
      local:本机
      localnet:本机所在网络地址；  
  注意：只能先定义，后使用；所以要写在option的前面
例： 
      acl localnet {
        172.18.134.134/16;
      };
 2. 内置的访问控制指令：
    allow-query {}; 允许查询的主机；白名单
    allow-transfer {}; 允许区域传送的主机；白名单
    allow-recursion {}; 允许递归的主机；一般仅允许本地网路地址做递归；白名单
    allow-update {}; 允许更新区域库内容；none 
```