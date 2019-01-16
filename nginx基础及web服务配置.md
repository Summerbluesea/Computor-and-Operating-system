
Nginx：

    NGINX is a free, open-source, high-performance HTTP server and reverse proxy, as well as an IMAP/POP3 proxy server. 

并发连接响应：

    httpd MPM：
        prefork：进程模型，两级结构，主进程master负责生成子进程，每个子进程负责响应一个请求；
        worker：线程模型，三级结构，主进程master负责生成子进程，每个子进程负责生成多个线程，每个线程响应一个请求；
        event：主进程master负责生成子进程，每个子进程响应多个请求；

    Nginx: AIO
        Nginx本身即为解决C10k开发，兼具事件驱动型IO和异步IO特性，一个进程能同时基于内部回调的方式处理多个请求；

Nginx特性：

    1. 模块化设计、较好的扩展性；
    2. 高可靠
    3. master/worker 
        master负责解析配置文件、管理worker进程，平滑版本升级；
        worker进程负责处理客户请求；
    4. 低内存消耗
        10000个keepalive模式下的connection，nginx只需消耗2.5M内存；
    5. 支持热部署；
        不停机更新配置文件、日志文件滚动、升级程序版本；

    6. 支持事件驱动机制，支持AIO， mmap；

Nginx基本功能：

    1. 静态资源的web服务器，能缓存打开的文件描述符；
        通过定义虚拟主机提供web服务；
    2. http, fastcgi, smtp/pop3/imap协议的反向代理服务器；支持缓存，代理缓存加速
   
    3. 七层负载均衡器(upstream模块)
        session保持
    4. Stream模块实现伪四层代理

Nginx扩展功能：

      1. 支持名称和ip地址定义的虚拟主机
      2. 支持keepalive
      3. 支持平滑升级
      4. 定制访问日志、支持使用日志缓冲区提供日志存储性能；
      5. 支持路径别名
      6. 支持基于ip及用户的访问控制
      7. 支持速率限制，支持并发数限制；


Nginx 安装和配置：

    安装：epel源安装或配置Nginx官方提供的repo进行安装；
    主配置文件：/etc/nginx/nginx.conf ； nginx安装后的默认配置是做为 web server

    Nginx的配置文件分为四个部分：

        全局配置；ngx_core_module中定义的所有可配置参数均在全局配置段定义
        web server or http reverse proxy(http配置)；所有ngx_http_*_modules
        smtp/pop3/imap协议的反向代理服务(mail配置)；所有ngx_mail_*_module
        伪四层负载均衡服务器(stream配置);所有ngx_stream_*_module
        

Nginx做为静态web server配置：

    1. worker_processes number | auto;
        worker进程的数量；通常应该等于小于当前主机的cpu的物理核心数；
        auto：当前主机物理CPU核心数；

    2. worker_rlimit_nofile NUMBER;
        worker进程所能够打开的文件数量上限;

    3. worker_cpu_affinity cpumask ...;
        worker_cpu_affinity auto [cpumask]; 定义worker进程与cpu核心数的亲缘性，绑定wokrer进程与cpu核心；

    4. worker_priority number;
        指定worker进程的nice值，设定worker进程优先级；[-20,20]

    5. 事件驱动相关配置
       event {
           worker_connections NUMBER  ;  每个worker进程可以响应的并发连接数；
       }

        Nginx可以响应的最大并发连接数：worker_processes*worker_connecctions 

    6. server虚拟主机
        用户的请求先通过匹配server_name被匹配到某一个server, 再根据URI匹配server中定义的某个location;
        server {
                    listen address[:PORT]|PORT;[default_server] [ssl] [http2 | spdy]
                    server_name SERVER_NAME;
                    root /PATH/TO/DOCUMENT_ROOT;

                    location [ = | ~ | ~* | ^~ ] URI {
                    root PATH
                    }
        }

        server_name name ...;
            指明虚拟主机的主机名称；后可跟多个由空白字符分隔的字符串；
                支持*通配任意长度的任意字符；server_name *.micro.com  www.micro.*
                支持~起始的字符做正则表达式模式匹配；server_name ~^www\d+\.micro\.com$

            匹配机制：
                (1) 首先是字符串精确匹配;
                (2) 左侧*通配符；
                (3) 右侧*通配符；
                (4) 正则表达式；

        location [ = | ~ | ~* | ^~ ] URI

            Sets configuration depending on a request URI.
            =：对URI做精确匹配；
            ~：对URI做正则表达式模式匹配，区分字符大小写；
            ~*：对URI做正则表达式模式匹配，不区分字符大小写；
            ^~：对URI的左半部分做匹配检查，不区分字符大小写；
            不带符号：以URI为前缀的所有uri；

            匹配优先级：=, ^~, ~/~*，不带符号；

    7. 基于ip地址的访问控制
       ngx_access_module 模块

        allow IP_ADDR
        deny IP_ADDR

        IP_ADDR：可以是主机ip，网络ip地址 172.18.0.0/16；缺省值表示allow all;

       可以用在 http, server, location, limit_except 上下文中；

    8. 基于basic账号认证
       ngx_http_auth_basic_module 

       借助于http-tools提供的htpasswd命令创建用户
        # htpasswd -c -m /etc/nginx/.ngxpasswd tom 

       location /images/ {
           root /www/html;
           auth_basic "private images";
           auth_basic_user_file "/etc/nginx/.ngxpasswd";
       }

    9. 路径别名
       alias， 只能用在location上下文 
       location /images/ {
           alias "/www/html/";
        }

        路径别名定义了location URI的另一个路径映射，用户访问location匹配到的URI映射至/www/html/目录；

    10. 错误页定义
       error_pages code = /code.html {
           root 
       }

    11. stub_status 

       ngx_http_stub_status；nginx状态信息输出；

       location = /status {
           stub_status;
       }

       Active connections: 1 
       server accepts handled requests
                19       19       23 
       Reading: 0 Writing: 1 Waiting: 0 

    12. 访问日志
       ngx_http_log_module

       access_log 
       log_format 
       open_log_file_cache;缓存日志打开状态

       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"'

       $http_x_forwarded_for： 如果客户端请求经由代理服务器代理而来，此变量表示真正的客户端ip；

       open_log_file_cache max=N [inactive=time] [min_uses=M] [valid=time];
                max:缓存的最大日志文件描述符数量；
                min_uses:在inactive定义的时间内最少访问量，小于此值则判定为无效缓存，予以清除；
                valid: 验证缓存项是否有效的时间间隔；

    13. 压缩功能
        ngx_http_gzip_module

        gzip on|off ;是否启用压缩
        gzip_comp_level ; 定义压缩级别
        gzip_proxied off | expired | no-cache | no-store | private | no_last_modified | no_etag | auth | any ...; 
          表示nginx是代理服务器在接收到被代理服务器的响应报文后，何种条件下启用压缩功能；
        gzip_types MIME_type; 定义压缩过滤器，对何种类型的MIME启用压缩；text/xml text/css application/javascript;

    14. ssl配置
        ngx_http_ssl_module 

        ssl on|off 
        ssl_certificate ; 证书路径
        ssl_certificate_key ; 服务器公钥
        ssl_protocols TLSv1 TLSv1.1 TLS1.2; 定义使用的ssl协议
        ssl_session_cache shared:ssl:10m; 
          build in: nginx内建的缓存，每个worker process独有自己的session缓存；
          shared: 所有的worker process 共享ssl_session缓存；
        ssl_session_timeout 10m;
        注意：由于https协议对maximun interoperability的限制，基于https协议的虚拟主机必须使用不同的ip地址；每个ip只定义一个https协议的server;

        # (umask 077;openssl genrsa -out nignx.key 2048)；生成私钥
        # openssl req -new -x509 -key nginx.key -out nginx.crt -days 3650 -subj "/CN=www.itunes.top"
        /etc/nginx/nginx.conf 
        server {
            listen 443 ssl;
            server_name www.itunes.top;
            ssl_protocols TLSv1.1 TLSv1.2;
            ssl_certificate /etc/nginx/ssl/nginx.crt;
            ssl_certificate_key /etc/nginx/ssl/nginx.key;
            location / {
            root /www/html;
            }
        }

    15. URL rewrite
        ngx_http_rewrite_module 

        The ngx_http_rewrite_module module is used to change request URI using PCRE regular expressions, return redirects, and conditionally select configurations.

        rewrite regex replacement [flag]；定义URL重写匹配条件和重写路径；
            如果定义的URL重写正则表达式匹配到请求的URI，请求的URI重写为replacement定义的重写路径；rewite指令按它们在配置文件中出现的顺序依次执行，可以定义flag终止其执行rewrite指令；

            如果relacement以"http://", "https://", or"$scheme"开始，进程终止同时返回客户端redirect URL给客户端；

            flag: last, break, redirect, permanent
                last:结束执行当前rewrite指令集，用重写后的URI进行新的location matching;
                break:停止执行当前rewrite指令集；
                redirect:返回temporary redirect 302 code给客户端；在重写路径不以"http://", "https://", or"$scheme"开始的情况下使用；
                permanent: 返回permanent redirect 301 code;

            location /images/ {
                    root /usr/share/nginx/html;
                    rewrite /images/(\.png$) /www/html/images/$1;
            }
            实现全站https访问,客户端访问http可以自动重写URL至https
            location / {
                rewrite /(.*) https://172.20.101.94:443/$1;
            }

    16. 合法引用者设置
        ngx_http_referer_module 

        valid_referers none | blokced | server_names | string ....; 定义请求报文的referer field合法可用值；
             none:没有referer首部；
             blocked:referer首部没有值；
             server_names: 请求报文的referer首部包含server_names定义的主机名；
                    arbitary string: 可以是首尾包含*通配符的字符串；
                    regular expression: 正则表达式匹配，要使用~打头；

            配置示例：
            valid_referers none block server_names *.itunes.top *.itunes.* *itunes.* itunes.* ~\.itunes\.;
                if($invalid_referer) {
                return http://www.itunes.top/invalid.jpg;
            }


