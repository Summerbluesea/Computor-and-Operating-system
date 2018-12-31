# DHCP 服务详解

DHCP，Dynamic Host Configuration Protocol，动态主机配置协议；
基于C/S架构，服务器端监听于67/udp端口，客户端监听于68/udp端口；
DHCP server在接收到客户端的请求时，分配可用的ip/netmask, gateway,DNS server等资源给客户端，DHCP server分配给客户端ip地址都有一个租期，客户端在此租期过半时需要向server发续租请求，如果因种种原因没有续租成功，到达租期后，server会收回此ip地址并标记为可用地址分配给其它客户端，DHCP服务可以缓解局域网内ip地址不够用的情况；

```bash
dhcp Client 获取地址过程：
 client发布dhcp-discover广播报文 -->  Server收到dhcp-discover报文，给客户端发送dhcp-offer，dhcp-offer报文中包含可用的ip/netmask,gw等信息 --> 
  Client收到server的offer, 决定用哪一个offer后，发送 dhcp-request 广播 --> Client采用的ip地址分配server发送ack报文给Client

dhcp Client在租期过半时向服务器续租过程
 Client向server发送dhcp-request单播报文 -->如果此ip仍可用，Server 发送dhcp-ack 给客户端完成续租
                                       -->如果续租server端回复dhcp-nack报文，拒绝续租；client则要发布discover广播，重新获取地址。
```
**DHCP的安装配置和使用**
```bash
DHCP程序是dhcp协议的开源实现，程序包名称dhcp，服务名称dhcpd
 /usr/sbin/dhcpd
  
 1. 安装
   yum install dhcp 
 2. 配置
   server端配置文件： 
   /etc/dhcp/dhcpd.conf 
   /etc/dhcp/dhcpd6.conf
  配置内容：
      option domain-name "domain.com" --指定搜索域
      option domain-name-servers 172.16.0.1, 8.8.8.8; -- 指定域名称服务器
      设置dhcp服务器提供的子网地址池信息
      subnet 172.16.0.0 netmask 255.255.0.0 {
          range 172.16.100.151 172.16.100.170;
          option routers 172.16.100.67;
          }
注意：此处subnet必须与主机在同一网段
3. 启动服务
 dhcp serverserver将分配出去的地址信息记录在/var/lib/dhcpd/dhcpd.leases文件中；

  lease 192.168.48.100 {
  starts 0 2018/12/02 13:08:10;
  ends 0 2018/12/02 19:08:10;
  cltt 0 2018/12/02 13:08:10;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 00:0c:29:39:b2:9b;
  client-hostname "centos6";

 dhclient得到的地址信息都记录在/var/lib/dhclient/dhclient-*.leases文件中；
  lease {
  interface "ens37";
  fixed-address 172.18.134.134;
  filename "pxelinux.0";
  option subnet-mask 255.255.0.0;
  option routers 172.18.0.1;
  option dhcp-lease-time 86400;
  option dhcp-message-type 5;
  option domain-name-servers 223.5.5.5,223.6.6.6;
  option dhcp-server-identifier 172.18.0.1;
  option domain-name "magedu.com";
  renew 1 2018/12/03 20:36:57;
  rebind 2 2018/12/04 08:18:12;
  expire 2 2018/12/04 11:18:12;

如果不想使用dhcp指定的dns服务，在网卡属性配置中PEER_DNS=no

```
**tftp服务**
```bash

tftp服务器实现

 1. tftp server 安装
  # yum install tftp-server 
 tftp服务依赖于守护进程xinetd，需先启动xinetd服务，再启动tftp服务
 2. 启动服务
  # systemctl start xinetd
  # systemctl start tftp-server
 3. tftp服务器配置文件
    /etc/xinetd.d/tftp
 4. tftp服务器的工作目录
    /var/lib/tftpboot
   tftp客户端可以上传文件至此目录，或从此目录下载文件
  
tftp客户端使用
 1. 安装
   # yum install tftp
 2. 运行
   # tftp tftp-serverIP
```
