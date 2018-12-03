# 系统加密和安全基础
## 加密解密基础
**加密类型及算法**
```bash     
 1. 对称加密：加密和解密使用同一个密钥；
    功用：加密数据
    特性：
      <1>. 加密/解密使用同一密钥
      <2>. 将原始数据分割成固定大小的块，逐个进行加密；
    缺陷：
      <1>. 密钥过多；每与一个用户通信，需要一个密钥；
      <2>. 密钥分发困难；
    算法：des, 3des, aes，blowfish, twofish
    工具：openssl enc, gpg 
 2. 公钥加密--公钥/私钥对，多用于密钥交换
    功用：
        数字签名:接收方确认发送方身份；发送方用自己的私钥加密数据，接收方用发送方的公钥解密，以确认发送方身份；
                数据完整性验证，单向加密算法提取数字特征码比对；
        密钥交换:发送方用对方公钥加密一个对称密钥给对方，完成安全密钥交换
        数据加密：公钥加密一般不用于数据加密，数据大使用的加密时间超长
    特性：用公钥加密的数据，只能使用与之配对儿的私钥解密；反之亦然
    算法：RSA(加密和数字签名)，DSA(数字签名)，ELGamal
    加密工具：openssl rsautl, gpg 
    
 3. 单向加密：只能加密，不能解密；多用于提取数据指纹，验证数据完整性
     特性：定长输出、雪崩效应;
     算法：md5, sha1, sha224, sha256, sha384, sha512
     工具：md5sum, sha1sum, sha224sum, sha256sum, sha512sum, openssl dgst
 4.  密钥交换：IKE
     实现方法一：公钥加密；
     实现方法二：DH算法加密
```
## OpenSSL命令行工具
**OpenSSL实现加密和解密**
```bash
1. OpenSSL实现对称加密
   加密：
  # openssl enc -e -des3 -a -salt -in testfile -out testfile.cipher
 解密：
  # openssl enc -d -des3 -a -salt –in testfile.cipher -out testfile
 
2. OpenSSL生成密钥对
  openssl genrsa 命令
生成私钥,并采用des3算法对私钥加密
   # (umask 077; openssl genrsa -out SECRET_KEY -des3 1024)
   对私钥解密
   # openssl rsa -in SECRET_KEY -out SECRET_KEY.bak
3 OpenSSL 实现单向加密
  # openssl dgst -sha512 FILE_NAME [-out /PATH/TO/SOMEFILE]；生成数字签名
 其它实现单向加密的工具：md5sum,  sha1sum, sha224sum, sha256sum, sha512sum
   # md5sum FILE_NAME [> /PATH/TO/SOMEFIEL]
4. OpenSSL 生成用户密码
   # openssl passwd -1 -salt SALT {PASSWORD}; -1指md5加密算法;
5. OpenSSL 生成随机数
  # openssl rand {-hex|-base64} NUM 
    NUM:指定随机数占用字节数；
```
## PKI 和 CA
PKI，全称为Public Key Infrastructure，公钥基础设施，PKI是为公钥加密技术提供的安全基础平台，负责管理密钥和证书；PKI通过数字证书的形式证明用户身份及他所持有密钥的正确匹配，是对公钥拥有者的身份证明；数字证书包括用户身份信息及用户的公钥；数字证书由权威的认证中心(Certification Authority)签发，CA使用自己的私钥为数字证书加上数字签名；任何想发放自己公钥的用户，可以向CA申请自己的证书，认证中心鉴定申请人的身份通过后，颁发利用自己私钥加密的包含自己数字签名和用户公钥等内容的证书给用户；用户以此通过PKI框架管理密钥和证书以建立安全的网络环境；
```bash
完整的PKI系统必须具有权威的认证机构(CA),数字证书库，密钥备份及恢复系统，证书作废系统等部分；
  CA：数字证书签发机关
  数字证书库：用于存储已签发的数字证书及公钥，用户可由此获得所需的其它用户的证书及公钥
  CRL：证书吊销列表

x.509是证书规范，其定义了证书的结构以及认证协议标准，符合x509规范的证书应包括以下内容：
    版本号
    序列号
    签名算法id
    发行者名称 --> 颁发此证书的CA名称
    有效期限
    主体名称 --> 证书拥有者名称
    主体公钥 --> 证书拥有者的公钥
    发行者唯一标识
    主体唯一标识
    扩展信息
    发行者的签名 --> CA签名, 提取以上数据的特征码并使用CA私钥加密后得到的即是CA签名
```
## SSL/TLS
SSL，Secure Socket Layer, 安全的套接字层，其是工作于应用层和传输层之间，SSL由Netscape公司发行，后在1999年，由ISOC接替Netscape在ssl基础上发行tls 1.0版，TLS全称为Transport Layer Security，为基于连接的传输层提供保密、通用、互操作、可扩展、高效率的协议，已成为互联网保密通信的工业标准，目前使用的是tls 1.2, tls 1.3于2018年发表；OpenSSL是一个开源项目，其中的一个组件是SSL/TLS协议的开源实现；
```bash
OpenSSL包括三个组件：
  openssl: 多用途的命令行工具；利用不同的子命令可以实现加密/解密，CA管理等；
  libcrypto:公共加密库；
  libssl: 库文件；ssl/tls协议的实现；如果某个应用层协议要利用ssl/tls协议实现加密通信，通过调用libssl库文件实现；

基于ssl/tls通信会话的建立：
(1) 通信双方在tcp三次握手后建立tcp会话
(2) 通信双方进行ssl handshake ：认证证书，协商加密算法
   以http服务为例：
    服务器端将自己的证书发给客户端，客户端验证服务器证书：
      -->1. 验证证书发行者是否受信任；客户端首先检查证书发行者的身份，去数字证书库找到CA的证书，从中提取CA的公钥，利用CA公钥解密server证书的数字签名，如果可以解密，则验证CA的身份可信；
      -->2. 检查server证书的主体名称与客户端访问的名称是否一致，如果不一致则不信任此证书；
      -->3. 利用证书中定义的签名算法计算得到证书的数字签名， 对比解密得到数字签名，如果一致，证书内容可信且完整；
      -->4. 检查证书是否在CA的吊销列表中；查看CA的crl, 确认证书在有效期内
      --> 完成以上内容，则证书认证完成
      -->5. 双方协商加密算法
(3)密钥交换
     Client用协商好的对称加密算法生成密钥，用server的公钥加密此密钥发送给server,完成密钥交换；
```
## 创建私有CA及颁发证书实现
```bash 
证书申请及签署步骤：
     1. 申请方生成证书申请请求并提交给注册机构；
     2. RA核验；注册机构验证申请方信息，如果验证通过，则提交CA签署证书；
     3. CA签署；
     4. 获取证书；
证书如果要在互联网使用则需向权威的认证机构CA申请；如果只是我们个人使用或公司内部网络环境使用，可以构建私有CA，下面介绍如何利用openssl命令行工具创建私有CA并颁发证书给用户;

创建私有CA配置文件: /etc/pki/tls/openssl.cnf ; 其中定义了私有CA各文件的存放路径

dir		= /etc/pki/CA		              # Where everything is kept
certs		= $dir/certs	         	    # Where the issued certs are kept
crl_dir		= $dir/crl	            	# Where the issued crl are kept
database	= $dir/index.txt    	    # database index file.
new_certs_dir	= $dir/newcerts	    	# default place for new certs.
certificate	= $dir/cacert.pem     	# The CA certificate
serial		= $dir/serial 		        # The current serial number
crlnumber	= $dir/crlnumber       	  # the current crl number  must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 	            	# The current CRL
private_key	= $dir/private/cakey.pem  # The private key

创建私有CA：
 进入CA目录树：/etc/pki/CA
(1)创建所需的文件
  # touch  index.txt 
  # echo 01 serial 
(2) 生成CA密钥对儿
  # (umask 077; openssl genrsa -des3 2048 -out private/cakey.pri )
(3) 生成自签名证书
  # openssl req -new -x509 -key private/cakey.pem -days 7300 -out cacert.file 
          -new:生成新证书签署请求
          -x509：专用于CA生成自签名证书；
          -key:生成证书使用的私钥文件；
          -days:证书有效期
          - out /PATH/TO/SOMEWHERE :指明证书存放路径
  [root@centos7 CA]#openssl req -new -x509 -key private/cakey.pri -days 3650 -out cacert.pri
 (4) 发证
      (a) 用到证书的主机生成证书请求；
        客户端生成自己的私钥
        # (umask 077; openssl genrsa -out /etc/httpd/ssl/httpd.key 2048)
        # openssl req -new -key /etc/httpd.key -days 365 -out /etc/httpd/ssl/httpd.csr
      (b) 把请求文件传输给CA；
      (c) CA签署证书，并把证书发给请求主机；
        # openssl ca -in /path/to/*.csr -out /path/to/*.crt -days 365 
```

## OpenSSH
```bash
 SSH协议，Sesure SHell，安全的远程登录协议，基于C/S架构，其客户端默认监听在tcp的22号端口；
 SSH协议版本：
  V1：基于CRC-32做MAC，不安全；
  V2：双方主机协议选择安全的MAC方式；基于DH算法做密钥交换，基于RSA或DSA算法实现身份认证；
   
OpenSSH:是ssh协议的开源实现；

  Client: ssh, scp, sftp
     window客户端：xshell, putty, secureCRT, sshsecureshellclient
  Server: sshd 

客户端组件：ssh
  ssh, 配置文件 /etc/ssh/ssh_conf
  使用格式： ssh [user@]SERVER [COMMAND]
            ssh -l user SERVER [COMMAND] 
              -p PORT:给出远程SERVER端监听的端口号，默认为22号端口；
              -X:支持X11转发
              -Y：支持信任的X11转发；
  客户端第一次访问ssh server，server同意连接后，客户端会自动复制自己的公钥至server端登陆用户家目录下.ssh/know_hosts文件中

openssh提供了两种客户端用户登陆认证方式：
    基于password验证
    基于key验证 
    
ssh客户端指定用户基于key验证登录：
 linux主机：
   (1)在client端生成密钥对
   # ssh-keygen  [-t dsa | ecdsa | ed25519 | rsa | rsa1] [-f keyfile]
     默认加密方式：rsa
     默认存储目录：/root/.ssh/id_ rsa
      Generating public/private rsa key pair.
      Enter file in which to save the key (/root/.ssh/id_rsa):
    (2)将公钥拷贝至server端
    # ssh-copy-id -i /home/USER/.ssh/id_rsa.pub [user@]SERVER_IP
    (3)server端[指定用户]用户家目录/home/USER/.ssh/authorized_keys下存放着client端的的公钥
    (4)启用代理程序，帮助client处理输入密钥口令
        # ssh-agent bash 
        # ssh-add [/PATH/TO/PRI_KEY]
windows客户端 
  (1) xshell客户端生成密钥对
  (2) 将公钥拷贝至server端的/home/USER/.ssh/authorized_keys文件中

Server: sshd 
  配置文件：/etc/ssh/sshd_conf 
  

常用参数：
  1. Port, 修改监听端口；默认为22
  2. listenaddress 修改监听地址；只允许哪一个ip或哪个网段的ip连接, 生产环境中只监听公司内部网络
  3. LoginGraceTime; 登陆宽限时长
  4. PermitRootLogin:是否允许远程以root用户连接
  5. MaxAuthTries;最多尝试验证次数
  6. MaxSession;
  7. PubkeyAuthentication yes:基于公钥认证登陆
     .ssh/authorized_keys
  8. PasswordAuthentiication yes: 基于口令认证登陆
  9. SyslogFacility AUTHPRIV  --> /var/log/secure ,经常分析日志，抓取登录失败尝试
  10.UseDNS no -- 远程连接时是否反解析ip地址

限制可登录用户的办法：在配置文件中设置
  AllowUsers user1 user2 user3
  AllowGroups group1 group2 group3 
```

