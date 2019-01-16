## 容器Container

Container:是一个独立的用户空间，程序本身和其依赖的运行环境，这个用户空间仅运行这一个进程；

docker: 将一个程序和其依赖的运行环境都做成了镜像文件；用户只需将镜像文件下载至本地， 由docker进程管理这些进程运行的独立的用户空间；

## docker 基础

OCI: Open Container Initiative 

由Linux基金会主导于2015年6月创立
围绕容器格式和运行时制定一个开放的工业化标准 
 two specification 
        - the Runtime Specification 
        - the Image Specification 
            -OCF: Open Container Format 

runC:  
命令行工具；用于根据OCI标准监控和运行容器；
每个容器以runC的一个子进程运行，也可以植入于不同的系统中而不需要运行一个单独的守护进程；
runC建立在libcontainer之上

docker architechture 
    C/S结构，客户端和服务器端运行在同一主机上，采用unix socket、基于http/https协议通信，共享内存；
 docker daemon(runC)
     监听Docker API的请求，管理docker objects(镜像文件，容器，网络，存储卷)

 docker client 
   - 是众多docker users与Docker进程交互的主要方式
        - 通过调用docker API执行docker command 
   
   - docker registries 
        - 用于存储docker镜像文件
        - docker hub 和 docker cloud 是公开的registries; docker的默认配置的registry指向docker hub 

docker objects 
当你使用docker时，就是在创建和使用images, containers, networks, volumes, plugins和其它objects;
   - images 
        - image是一个只读的模板，用于创建docker container;
        - 分层构建，一个镜像创建在另一个镜像基础上；
        - 可以自己创建镜像或使用registry上其他人发布的镜像；
    
    - containers 
        - 一个容器是一个镜像文件的运行实例；
        - 你可以通过Docker API 或者CLI创建、运行、停止、移动或删除容器
        - 可以将容器连接至一个或多个网络，attach storage to it, 也可以基于它现在的状态创建镜像；
    - networks 
        - docker在创建或启动运行一个容器时，可以指定容器使用的网络；
        - 默认由docker创建的容器均使用docker0桥的nat网络中
    - volumes

**安装和使用docker** 

- 依赖的基础环境
    - 64 bits CPU
    - linux kernal 3.10+
    - linux kernal cgroups and namespace 

- CentOS 7 
    - "Extras" repository 

- 官方站点 
  www.docker.com 
- 安装
  下载安装国内docker镜像站点提供的docker-ce.repo 

  yum install docker-ce -y 

  采用镜像加速器访问docker的registry 

   \# mkdir /etc/docker

   \# vim /etc/docker/daemon.json 
   
   \# systemctl daemon-reload 
   \# systemctl start docker 

- 命令格式
```bash 
 docker [OPTIONS] COMMAND

 Management Commands:
  builder     Manage builds
  config      Manage Docker configs
  container   Manage containers
  engine      Manage the docker engine
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

查看每个管理命令的子命令 # docker MANAGERMENT_COMMAND -h 

注意：非以守护进程运行的镜像，必须指定为交互式运行；
      docker启动运行的container都加入至docker0的nat网络中；
      安装启动docker之前，把firewalld关闭；
      容器内唯一运行的程序一定要运行在前台
```
**Docker Container**
```bash
创建容器：
   docker [container] create 
   docker [container] run 
     -t,--tty . 容器运行时提供一个终端
     -i,--interactive ；容器运行时提供一个交互式接口
     --name string 
     --network 
     --rm 容器停止并自动删除，not compatible with -d
     -d,--detach

查看运行状态容器的信息：
    docker [container] ps 

接入容器内执行指定命令：
    docker container exec CONTAINER COMMAND [args...]

    如果容器运行的是一个以守护进程方式前台运行的程序，可以使用命令dcoker exec CONTAINER -it /bin/sh；进入/bin/sh提供的与容器的交互式接口


获取容器控制台日志内容：
    docker [container] logs CONTAINER
查看container进程系统资源使用情况：
    docker [container] stats
    docker top CONTAINER 
剥离运行容器与当前终端：
    ctrl+p, ctrl+q 
    重新将运行容器关联至当前终端：
    docker attach CONTAINER

```
**Docker images**
```bash
docker镜像包含启动容器所需要的文件系统及其内容，用于创建并启动docker容器；
  - 分层构建 
     - 最上层为“可读写”层；创建容器时创建；
     - 其下均为“只读”层；由镜像提供，可由不同的container共享；
     - 建立在宿主机指定目录下的文件系统之上
        - Graph Driver
            aufs: Advanced multi-layered unification filesystem 
            overlay2 
  - 联合挂载 
     - CoW ；存在于只读层的文件在第一次需要修改时复制到“读写层”进行修改；
             Container用户看到的文件是多层叠加结果；(上层有的文件会覆盖下层)

docker registry
   - docker hub, docker cloud
  
   - private registry 

registry上提供众多repositories, 每个repository包含一个程序迭代版本的镜像文件;
  dockerfile推送至github的repository, docker hub会自动检测到dockerfile并基于其创建一个new image;
   - 顶层repository
   - 属于个人或组织的repository ； 
   - 在docker hub 上建自己的仓库； 

 查看所有images
    docker image ls 
   
 查看指定镜像的详细信息
    docker image inspect

如何自己做镜像：
   1. 基于容器做镜像
     docker container commit CONTAINER [REPOSITORY[:TAG]]; 将当前“读写层”修改的数据保存为镜像，基于现在容器运行的镜像创建了一个新的镜像；
       -a,--author
       -c,--change-list ; 修改镜像中指定容器启动运行的默认程序为指定程序

        # docker container commit -p -a "Huomei <huojunmei_1@163.com>" -c "CMD ['/bin/sh', '-c',/bin/httpd -f -h', '/data/web/html']" b2  huomei/test:v0.2

        登录docker hub，可以将镜像推送至自己建的repository
        # docker login
        # docker image push -h
        # docker tag huomei/test:v0.2 huomei/test:latest 

   2. 基于某个镜像文件，在Dockerfile中定义操作，利用docker image build工具制作镜像
      - Dockerfile是一个文本文件，包含构建镜像的所有指令；
      - docker image build 通过顺序执行Dockerfile中的指令自动创建镜像；
      - 需要在一个工作目录进行，Dockerfile就放置在此目录下，所有构建镜像需要的文件均放置在此目录下；

      Dockerfile中的环境变量：
         
        申明环境变量：ENV variable_name 
        可以在申明变量之后引用变量：$variable_name 或 ${variable} 或 ${variable_name:-word} (如果变量没有赋值，则默认值为word)

        环境变量：一个docker镜像为了适用于不同运行环境而定制的环境变量，可以基于环境变量的不同赋值使得应用程序实现的不同运行特性；
                  实现基于单一镜像启动的程序可以有不同的运行特性;

                  docker image  --entrypoint.sh脚本传递参数--> 定制程序的运行特性
            
            - entrypoint.sh脚本 
                 - entrypoint.sh脚本是容器启动初始化脚本，执行完成后退出，退出前启动容器镜像中定义的默认程序；
                 - 将用户传递的参数替换配置文件中的环境变量？？

       Dockerfile Instructions 

    	- FROM 
    		- Syntax 
    		    FROM <repository>[:tag] 或 FROM <repository>@<digest> 

    	-LABLE
    		- Syntax 
    			LABLE <key>=<value> <key>=<value>
    		- adds metadata to the image 

    	- COPY 
    		- 从docker正在运行的一个Container复制文件至创建的新映像文件
    		- Syntax 
    			COPY <src> ...<dest> 或 COPY ["<SRC>"..."<dest>""]
    		- 所有复制的文件必须放置在工作目录下，起始于工作目录
    		- docker build parent_dir_of_dockerfile -t name:tag 

    	- ADD 
            - 类似于COPY指令，ADD支持使用tar文件和url路径
            - 如果<src>为URL且<dest>不以/结尾，则<src>指定的文件将被下载并直接重命名为<dest>；如果<dest>以/结尾，则     URL文件将被下载并保存在<dest>/目录下；
            - <src>是本地文件系统上的tar文件，将被展开至指定<dest>目录下一个目录；而通过URL获取到的tar文件将不会被自动展开；
    	- WORKDIR 
    		- 用于为所有的RUN, CMD, ENTRYPOINT, COPY和ADD指令设定工作目录
            - 指定容器中的目录，可以通过环境变量赋值 

    	- VOLUME 没听懂！！！ 
    		- 创建镜像挂载点，当作容器运行的存储卷；挂载时会先将挂载点上的文件复制到存储卷，再挂载；

    	- EXPOSE 
          - 用于为容器打开指定要监听的端口以实现于外部通信；
    	  - 容器直接共享宿主机的网络namespace； 
    	  - 把容器的网络地址expose DNAT规则
          - EXPOSE [<port>/<protocol>] [<port>/<protocol>]
          - EXPOSE定义端口需在运行docker镜像时使用-P选项，暴露EXPOSE指定端口

    	- ENV --> 在build阶段使用的环境变量
    	   - 可以在dockerfile中位于其后的其它指令调用

    	- ARG --> dockerfile新版支持环境变量
           - 执行docker build时传递参数赋值给Dockerfile中ARG指令定义的变量
    	- RUN 
    	    - 制作镜像时运行的命令，可以是基础镜像中存在的任何命令；
            - RUN 指定的命令，在docker build过程执行，所以必须是基础镜像存在的命令；
    	- CMD 
    		- docker run container ;基于镜像运行container时默认运行程序；
            - 可以在docker run 命令时指定其它命令覆盖CMD定义的默认运行目录；
            - CMD SHELL COMMAND 
            - CMD ["COMMAND","ARGS"]
    	- ENTRYPOINT
    		- 用于指定基于镜像运行container时默认运行程序 ；
            - 使用ENTRYPOINT指定默认运行程序不能被覆盖docker run命令行选项覆盖
            - 通过docker run命令行传递--entrypoint args 参数修改默认运行命令；
               #docker run --name web2 -it --rm --entrypoint "/bin/sh" php-httpd:v0.6 ；web2启动后的运行程序为/bin/sh, 而非ENTRYPOINT指定的默认程序/usr/sbin/httpd -DFOREGROUD
    		- CMD和ENTRYPOINT同时指定，CMD指定的命令都是ENTRYPOINT的参数；
            - 一般ENTRYPOINT处运行一个脚本entrypoint.sh，用于通过环境变量赋值为CMD指定的运行程序提供配置；
              docker run 命令通过选项-e传递参数给环境变量赋值
            # docker run --name web -it --rm -P -e DOC_ROOT=/web1/bbs -e LISTEN_PORT=9527 web:v0.4
              ENTRYPOINT ["/bin/entrypoint.sh"]

               entrypoint.sh脚本 -->基础镜像支持的shebang
               #！/bin/bash
               #
               servername=${SERVER_NAME:-lovalhost}
               docroot=${DOCROOT:/var/www/html}

               cat > /etc/httpd/conf.d/myweb.conf <<EOF
               <Virtualhost *:80>
                    ServerName "$servername"
                    DocumentRoot "$docroot"
                    <Directory "$docroot">
                            Options none 
                            AllowOverride none
                            Require all granted 
                    </Directory>
                <Virtualhost>   
                EOF
                exec "$@" --> 用目标命令(CMD指定的命令)替换当前运行的shell

        - HEALTHCHECK  
            - 容器运行健康状态检测指令
            - options 
               --interval=(default 30s)
               --timeout=(defualt 30s)
               --start-period=(default 0s)
               --retries=(default 3) 连续三次检测为Unhealthy则显示unhealthy,需借助容器编排工具实现自动重启容器；
        - ONBUILD 
            - 定义基于现在定义的镜像做镜像文件时运行的INSTRUCTION 
    ```
 
      
	镜像分发方式：
	   1. repository ；docker hub 
	      pull至自己建的仓库，并公开给他人访问
	   2. docker image save /dir/image.tar

	      docker image load /path/from/image.tar
```
**Docker registry** 

   官方提供的镜像registry: docker hub 
   第三方提供的registry: 阿里云为客户提供的registry; quay.io
   私有registry:

registry包括两个部分：
   1. repositories  

     每个repository由特定的docker镜像的所有迭代版本组成
   2. index 
     维护用户账户、镜像的校验以及公共命名空间的信息

如何创建私有registry：
```bash 
方法 1. docker-distribution建立registry
        安装: yum install -y docker-distribution 
        配置：/etc/docker-distribution/registry/config.yml
        启动：systemctl start docker-distribution ; 监听在5000/tcp , http协议通信
    默认情况下，docker-distribution提供的regidtry不支持划分名称空间，也不支持用户认证机制;
    向此registry上传镜像时要给标识镜像tag:
       tag 格式：  HOST_ip:5000/仓库名:tag 
    
    默认情况下，docker不接受http协议明文传输的镜像，所以要在docker配置文件配置，并重启docker服务
     /etc/docker/daemon.json 
     "insecure_registry": [""] 

方法 2：harbor 
      基于docker-distribution的二次开发，开源的，实现企业级private registry;
      官网：goharbor.io 
```      

**Docker Networks**

Docker为其管理的容器提供四种网络，可以在创建容器时指定；
   - 桥网络：bridge 
        - docker0 NAT
        - bridge container
   - 共享桥，联盟式网络 
        - 几个container之间共享其中一个容器的网络名称空间；容器间可以通过127.0.0.1通信；
        - 每一个容器有自己独立的mount, PID, USER 
        - 容器间共享net, IPC, UTS 
        - bridge container
   - 共享host网络
        - 容器共享宿主机的网络名称空间；
        - 以守护进程方式运行在容器中的程序，但是实现系统级管理任务时使用；
        - 容器共享宿主机net,IPC和UTS
        - open container 
   - none 网络
        - 容器运行的进程无需用到网络时使用
        - closed container

    - 也可以手动创建网络：docker network create 

    - 将容器连接到某个网络上：docker network connect NETWORK CONTAINER
```bash 示例：

创建并运行一个容器，指定其网络为none 
 # docker run --name myweb  --rm --network none nginx:1.15-alpine 

容器joinweb共享容器myweb的网络名称空间：net, IPC, UTS
# docker run --name myweb -it --network bridge --rm nginx:1.15-alpine
# docker run --name joinweb -it --rm --network container:myweb nginx:1.15-alpine  /bin/sh

创建容器myweb共享宿主机的网络名称空间
# docker run --name myweb --network host nginx:1.15-alpine 


其它网络相关参数设定：设定主机名、dns server等内容
 
    可以在创建或运行容器时指定容器的主机名、生成/etc/hosts文件，完成本地域名解析；
    # docker run --name 
    或者指定容器的DNS服务器、搜索域等内容(生成/etc/resolv.conf文件内容)；
    #docker run --name box1 -it --rm --hostname box1.localhost --add-host ftp.magedu.com:172.18.0.1 --dns 172.18.0.1 --dns 8.8.8.8 --dns-search ilinux.io busybox
    / # hostname
    box1.localhost
    / # cat /etc/hosts
    127.0.0.1	localhost
    172.18.0.1	ftp.magedu.com
    172.17.0.2	box1.localhost box1
    / # cat /etc/resolv.conf
    search ilinux.io
    nameserver 172.18.0.1
    nameserver 8.8.8.8
```

如果容器使用docker0的NAT桥网络，宿主机自动生成了docker0的SNAT规则，容器可以访问外部网络，但外部网络主机访问不到宿主机内的容器，需手动添加DNAT规则，通过访问宿主机的指定ip地址和端口实现容器提供的服务；

   - 将宿主机的指定ip地址的访问全部映射给容器的ip
     iptables -t nat -A PREROUTING -d 宿主机ip -j DNAT --to-destination 容器ip 
     
   - 将对宿主机某ip的某端口访问映射给容器的某ip的某端口
      iptables -t nat -A PREROUTING -d 宿主机ip -p {tcp|udp} --dport 宿主机端口 -j DNAT --to-destination 容器ip:port

使用docker run命令创建并运行容器时，可以使用-p选项，给出宿主机ip:port与容器ip:port的映射关系，会自动生成DNAT规则；

```bash
 -p 选项使用格式:
    -p <containerPORT>; 将容器的指定端口映射给宿主机所有ip的一个随机端口
    -p <hostPort>:<containerPort>; 将容器端口映射给宿主机指定端口；
    -p <ip>::<containerPort>;将容器端口映射给宿主机指定ip的随机端口上
    -p <ip>:<hostPort>:<containerPort>; 将容器端口映射给宿主机指定ip的某端口上；
 -P 暴露容器所有监听端口
```

```bash
自定义docker0桥的网络属性：/etc/docker/daemon.json 
 .json格式的配置文件，每一项都以“,”结束，但最后一行定义结束不加“，”；
 {
     "bip": "172.31.0.1/16"
 }
 bip定义docker0桥的ip地址，连接至此桥的容器自动获取docker0桥所在网段地址；

```

**Docker Data Volumes** 
类似于在创建并运行容器，将容器的”可写层“直接挂载至宿主机文件系统上，这样容器的数据可以实现持久化存储；不会随容器生命周期的结束而被删除；
```bash
Data volume
   - volumes是容器上的一个或多个“目录”，此类目录可绕过联合文件系统，与宿主机的某目录“绑定(关联)”；
   - volumes于容器初始化时创建，由base images提供的卷中的数据会于此期间完成复制；
   - container之间可以共享和重用volumes;
   - volumes中的数据独立于容器的生命周期；

Volume types 
   - Bind mount volume 
     docker daemon创建存储卷指向宿主机文件系统一个user-specified位置；docker run命令创建并运行容器时需指定volume名称以及在宿主机上的绑定目录；
              
   - Docker-managed volume 
     docker daemon创建的存储卷，其绑定在宿主机文件系统属于docker进程的目录下；docker run命令创建并运行容器时，只需指定volume的名称即可，宿主机上的绑定目录会自动生成；

在容器中使用volume 
   使用docker run 命令创建并运行一个容器时，使用-v 选项
   - Bind-mount Volume 
       -v HOSTDIR:VOLUMEDIR 
   - Docker-managed volume 
       -v /volumedir 

容器的存储卷绑定于宿主机的同一个目录下，实现文件共享：
 # docker run --name box2 -it --rm -v /tmp:/data/box2 busybox
 # docker run --name box1 -it --rm -v /tmp:/data/volumes busybox

或者 新创建并运行的容器直接复制要共享容器的存储卷路径：
 # docker run --name box2 -it --rm --volumes-from box1 busybox
```bash

Docker构建AMP:
#  docker run --name web1 --rm -v /tmp/web:/var/www/html -p 80:80 php:5.6-apache
# tar zxf wordpress-5.0.2.tar.gz -C /tmp/web/
# docker run --name dbsrv -e MYSQL_ROOT_PASSWORD=123456 --rm -v /tmp/data:/var/lib/mysql mariadb:10.2
# docker run --name dbsrv -it --rm -e MYSQL_ROOT_PASSWORD=123456 mariadb:10.2
```
## Docker-compose基础应用
Docker自己提供的容器编排工具，使用python开发；
```bash 
命令行工具：docker-compose 
配置文件：docker-compose.yml，必须放在docker-compose的工作目录下，

docker-compose按docker-compose.yml中编排的指令运行并配置容器；

docker-compose.yml文件格式：
version: "3" --> 表示docker-compose.yml格式的版本
services:    --> service定义启动的容器；每个容器要分别定义容器镜像文件、网络、启动几个容器、容器重启策略等
