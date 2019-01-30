## Java基础

Java是一种完全面向对象的应用级编程语言，由Sun公司研发，早期是为数字电视机顶盒应用程序开发，1994年，随着http协议的发展，应用程序有了动态网页需求，java应用于网站应用程序的开发而得到发展，java的发展简史如下：

    Java起源于Sun公司的Green Project项目, Oak,  James Gosling;
        1995：Java 1.0, Write once, Run Anywhere;
        1996：JDK（Java Development Kit），包含类库(API)、开发工具(javac)、JVM（SUN Classic VM）  
              JDK 1.0,  Applet, AWT
        1997：JDK 1.1
        1998: JDK 1.2
        1999：Sun分拆Java技术为三个方向：
                Java1.#系列的版本称为J2;
                    J2SE：Standard Edition；  包含的类库适用于开发单机应用程序；
                    J2EE：Enterprise Edition；包含企业级分布式通信开发需要的类库；
                    J2ME：Mobile Edition；    移动端应用程序开发需要的类库；

        2000：JDK 1.3
              HotSpot VM (Sun收购HotSpot，Java虚拟机的另一个实现)
        2002：JDK 1.4
        2006：Java 1.6, Sun开源了Java技术，GPL，建立一个称为OpenJDK组织；
        2011：JDK 1.7
        2014：JDK 1.8
        2016：JDK 1.9
        2017: Java 10
        2018: Java 11


Java技术体系(JDK组成部分)：是一个J2SE的标准环境；

    Java Language
    Tools&Tools APIS:调试工具；
    JRE: Java Runtime Enviroment，包含JVM和Class Libiary；

        Java language:完全面向对象的编程语言；
        Class File Format: 类文件格式；
        Class libary: 类库；把基础的功能代码化，可以直接调用使用；
        JVM：java程序运行接口，为java编写的程序提供运行环境，可以隐藏底层操作系统的差异性；

Java服务器程序页面jsp：Java Server Page;
    applet:是一个java类库，客户端应用程序调用；

    servlet: 一个java类库，服务器端应用程序调用servlet类库，为jsp程序运行提供基础架构，提供处理通用协议的功能；

    JSP: 在servlet基础上又加了jsp类库；可以将非java程序片段转换为java类文件格式；J2EE的类库组件；

        jsp程序文件执行：*.jsp；

            首先由JSP类库中的Jasper组件预处理，把非Java代码片段转化为java类文件格式的打印输出，从而把整个文件全部java代码化，变为.java格式；

            然后交由JavaC编译器编译成java类文件格式，.class;

            再交给Servlet，通过调用JVM运行程序代码，将结果返回；

Java 2 EE的商业版实现：
    JSP类库：Java Web Server

    IBM：WebSphere
    BEA: WebLogic
    Oracle: Oc4j
    RedHat: Jboss 

JSP+servlet开源实现：

    JWS：Java Web Server，Sun 给出的JSP+servlet的参考性实现；

    Tomcat: ASF也开发了一款软件实现JSP+servlet类库，名为Jserv;后来Sun把JWS捐给了ASF；ASF结合JWS重构了JServ，项目称为Catalina;Catalina面世后广受欢迎，O'Reilly出版社出版Catalina项目的书籍封面是动物猫tomcat，后来ASF就使用tomcat命名此项目了；

## Tomcat

上面也介绍了tomcat由来，tomcat是用java编程语言写的JSP Servlet类库的实现；tomcat自身也是java程序，也需要运行在JVM之上，其可以监听于套接字上，但自身不提供服务，真正服务于用户请求的是部署在tomcat上的网页站点程序；

tomcat监听的套接字对外支持3种协议：

    http
    https
    ajp: Apache Jserv protocol;ajp协议只能用于与httpd进程通信；是一种二进制协议；

tomcat实例组件：

    每一个组件都是一个类的实例化对象；
    server: connector
        service:
            connector: 监听套接字组件；接收用户请求；
            service: 一个service的所有connector关联至一个engine上
            engine: 一个connector只能服务于一个engine；真正容纳jsp程序的组件；

                host: 定义虚拟主机，一个engine内可以定义多个虚拟主机；

                    context: 定义一个jsp应用程序上下文；host内可以定义多个context;从而实现每一个context可以被单独部署和管理；

java程序的部署(deploy)：

    所谓部署，就是将java程序及其依赖的类库编译并加载至JVM并实例化为对象准备就绪等待客户端访问；
    每一个应用程序都可以被单独启动、停止、重启、反部署;应用程序的部署描述文件web.xml文件；

**tomcat安装启动**

    1. 部署JDK

    要与程序员开发所用的JDK版本适配;

    (1)	OpenJDK
    
        redhat支持多版本并存，可以使用alternative命令设置默认运行版本，用哪一个调用哪个就可以；

        # yum install java-11-opjdk-devel

    (2) Oracle官方提供的JDK rpm包安装

        # rpm -ivh jdk-11.0.2_linux-x64_bin.rpm 
        程序安装在/usr/java/ 目录下，支持多版本并存，可以使用default和latest定义默认运行和最新版本；

    定义环境变量：
        JAVA_HOME=/usr/java/default
        PATH=$JAVA_HOME/bin:$PATH

    修改默认运行的jdk版本：

    alternatives --config java



    2. 安装tomcat 

    (1) rpm包安装
        # yum install tomcat-admin-webapps tomcat-docs-webapp tomcat-webapps -y


        配置文件：
            /etc/tomcat/context.xml
            /etc/tomcat/server.xml
            /etc/tomcat/tomcat-users.xml
            /etc/tomcat/tomcat.conf
            /etc/tomcat/web.xml   -->定义tomcat部署描述信息，定义程序有哪些依赖的类库

        service unit文件：/usr/lib/systemd/system/tomcat.service

        启动：systemctl start tomcat 

    (2) 官方提供的二进制程序包

        # tar xvf apache-tomcat-8.5.37.tar.gz -C /usr/local/
        # cd /usr/local
        # ln -sv apache-tomcat-8.5.37 tomcat
        # useradd tomcat
        # chown -R tomcat.tomcat tomcat

        启动：
        # bin/catalina.sh start
  
 
        tomcat的目录结构
                bin：脚本，及启动时用到的类；
                conf：配置文件目录；
                lib：库文件，Java类库，jar；
                logs：日志文件目录；
                temp：临时文件目录；
                webapps：webapp的默认目录；
                work：工作目录；

**tomcat配置**

tomcat的配置文件构成：

    server.xml：主配置文件；
    web.xml：每个webapp只有“部署”后才能被访问，它的部署方式通常由web.xml进行定义，其存放位置为WEB-INF/目录中；此文件为所有的webapps提供默认部署相关的配置；
    context.xml：每个webapp都可以专用的配置文件，它通常由专用的配置文件context.xml来定义，其存放位置为WEB-INF/目录中；此文件为所有的webapps提供默认配置；
    tomcat-users.xml：用户认证的账号和密码文件；
    catalina.policy：当使用-security选项启动tomcat时，用于为tomcat设置安全策略； 
    catalina.properties：Java属性的定义文件，用于设定类加载器路径，以及一些与JVM调优相关参数；
    logging.properties：日志系统相关的配置；


JSP Webapp的组织结构：
    /:webapps的网页文件根目录
        index.jsp, index.html：主页
        WEB-INF/: 当前webapp的私有资源路径；通常用于存储当前webapp的web.xml和context.xml配置文件；
        META-INF/: 类似于WEB-INF/;
        classes/: 类文件，当前webapp提供的类；
        lib/: 类文件，当前webapp所提供的类，被打包为jar格式；

在tomcat上部署(deploy)webapp的相关操作：

    deploy：将webapp的源文件放置于目标目录(网页程序文件存放目录)，配置tomcat服务器能够基于web.xml和context.xml文件中定义的路径来访问此webapp；将其特有的类和依赖的类通过class loader装载至JVM；

        部署有两种方式：
            自动部署：auto deploy
            手动部署:
                冷部署：把webapp复制到指定的位置，而后才启动tomcat；
                热部署：在不停止tomcat的前提下进行部署；
                    部署工具：manager、ant脚本、tcd(tomcat client deployer)等；
    undeploy：反部署，停止webapp，并从tomcat实例上卸载webapp；
    start：启动处于停止状态的webapp；
    stop：停止webapp，不再向用户提供服务；其类依然在jvm上；
    redeploy：重新部署；


tomcat配置：server.xml

tomcat的常用组件配置：

    Server：代表tomcat instance，即表现出的一个java进程；监听在8005端口，只接收“SHUTDOWN”。各server监听的端口不能相同，因此，在同一物理主机启动多个实例时，需要修改其监听端口为不同的端口； 

    Service：用于实现将一个或多个connector组件关联至一个engine组件；

    Connector组件：端点
        负责接收请求，常见的有三类http/https/ajp；

        进入tomcat的请求可分为两类：

            (1) standalone : 请求来自于客户端浏览器；
            (2) 由其它的web server反代：来自前端的反代服务器；
                nginx --> http connector --> tomcat 
                httpd(proxy_http_module) --> http connector --> tomcat
                httpd(proxy_ajp_module) --> ajp connector --> tomcat 
                httpd(mod_jk) --> ajp connector --> tomcat 

    1. tomcat实例管理端口
        <Server port="8005" shutdown="SHUTDOWN">

        可以关闭监听管理端口 Server port="-1"
    
    2. 虚拟主机配置

         <Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true"/>

         name:定义虚拟主机主机名；
         appBase: 网页文件根目录；可以是绝对路径也可以使用相对TOMCAT_HOME相对路径；
         unpackWARs: 是否自动展开.war文件；
         autoDeploy: 此虚拟主机网页文件根目录下的程序是否自动部署；

        <Host name="www.blog.top" appBase="/myapps" unpackWARs="true" autoDeploy="true" >

        </Host>

         [root@centos7 myapps]#tree
            .
            └── ROOT
                ├── classes
                ├── index.jsp
                ├── lib
                ├── META-INF
                └── WEB-INF

    3. 在同一虚拟主机中部署不同的JSP应用程序
        通过context组件实现：

        <context path="/app" docBase="/data/apps" reloadable="" />

    4. Valve组件：

        Valve存在多种类型：
            定义访问日志：org.apache.catalina.valves.AccessLogValve
            定义访问控制：org.apache.catalina.valves.RemoteAddrValve

        <Valve className="org.apache.catalina.valves.RemoteAddrValve" deny="172\.16\.100\.67"/>

    5. tomcat的两个管理应用:
		manager apps：管理webapps应用程序; 提供manager gui web接口
		host-manager：管理虚拟主机

    不成功！
        基于ip地址的manager web访问控制:
         <Context path="/manager" docBase="/usr/share/tomcat/webapps/manager" reloadable="" >
            <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.0\.0\.1" />
        </Context>

        基于ip地址host-manager访问控制
        <Context path="/host-manager" docBase="/usr/share/tomcat/webapps/host-manager" reloadable="" >
            <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.0\.0\.1" />
        </Context>




