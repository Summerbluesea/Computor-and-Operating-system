## Nginx做web服务反向代理服务器配置

  **ngx_http_proxy_module**

    1. proxy_pass URL ;
        Sets the protocol and address of a proxied server and an optional URI to which a location should be mapped.可以用于location, if in location, limit_except上下文；

        As a protocol: http or https:
            proxy_pass http|https://FQDN|ip_addr[:port]/uri/;

        Or as a UNIX-domain socket:
            proxy_pass http://unix:/path/to/unix.sock:/uri/; 

        location /uri {
            proxy_pass http://172.20.101.94;
        }
        如果location定义的uri结尾没有"/"，客户端访问的uri相当于访问proxied server的网页文件根目录；
        如果location定义的uri结尾有"/", 客户端访问的uri相当于访问proxied server的网页文件根目录下补全uri路径；

        当location定义的uri使用正则表达式，proxy_pass的URL不能给定任何URI；前端请求的任何uri都会补全至proxy_pass的URL之后；
        location ~* \.(jpg|png|jpeg)$ {
            proxy_pass http://172.20.101.94;
        }

    2. proxy_set_header field value;
        nginx做为代理服务器可以修改请求报文首部信息，此选项就是设定发往后端proxied server的请求报文首部信息；

        proxy_set_header X_Real_IP $remote_addr;(是不是只适用后端web server是httpd)
        proxy_set_header X_Forwarded_For  $proxy_add_x_forwarded_for;(httpd和nginx都可以用)

        后端proxied server的访问日志记录也可以引用此处设定请求报文首部信息，例如：

          (1)  后端web server如果是httpd,后端服务器日志文件可以这样引用：
               Log_format  '"%{X-Real-Ip}i"' 或'"%{x-forwarded-for}"'
          (2)  后端web server是nginx, 则直接引用变量 $http_x_forwared_for

    3. nginx做web服务反向代理服务器缓存功能配置；
       注意：
        (1) nginx做http服务反向代理服务器缓存配置和做fastcgi反向代理服务器缓存配置是独立分别定义的；
        (2) nginx的缓存需先定义缓存空间，然后在代理上use；

        缓存的功能：
           由于程序存在时间和空间局部性，这表现为数据的热区特性，如果把热区数据放于缓存中，可以显著提高服务对客户端请求的响应速度；
        
        缓存数据的组织格式：
            key-value键值存储;

            key:用户请求url，取URL中的部分(uri)的hash值做为层级索引使用；代理服务器可以自定义索引层级；
            value:客户端响应报文内容

        反向代理服务器缓存：代理服务器本地存储，实现缓存加速；

        nginx做http反代的缓存设置：
            此类缓存也称为page cache；
            nginx将key和value分开存放，key存放于内存中，value存放于磁盘上；nginx关机后在下一次启动后可以根据磁盘上的value值重建内存中的key;

            定义缓存：proxy_cache_path ；用于http上下文；
            调用缓存：proxy_cache ; 可用于http,server,location上下文；

           (1) 定义缓存：proxy_cache_path
                proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];

                path:定义缓存的位置
                levels:定义缓存索引层级；例如levels=1:1:2, 表示从key的索引层级为3层，每一层的数字表示hash值的尾部的第一个十六进制数字开始取值，数字是几就取几个十六进制数做索引；
                keys_zone:定义缓存名称，调用缓存时使用；缓存的内存空间大小；

            配置示例：
                proxy_cache_path /var/nginx/cache levels=1:1:2 keys_zone=webcache:10m max_size=2G;

           (2) 调用缓存：proxy_cache

                proxy_cache 缓存名称；
                proxy_cache_key ; 定义缓存使用key，$scheme$proxy_host$request_uri，可以自己定义选择哪写部分；
                proxy_cache_valid [code...] time; 定义缓存有效时间
                proxy_cache_mathods GET|HEAD|POST...;根据请求方法缓存并查询缓存，可以实现只缓存读请求(GET HEAD);
                proxy_cache_use_stale ;定义请求发生错误时或代理服务器无法连接后端时缓存可用的；
                proxy_cache_purge $purge_method; 定义缓存清理方法；
                proxy_hide_header field；隐藏后端服务器响应报文某些首部信息，不显示给客户端；
                proxy_connect_timeout ;表示代理服务器向后端请求建立连接的超时时长；超时代理服务器会返回502，bad gateway;
                proxy_read_timeout ; 等待后端服务器的响应报文的超时时长；
                proxy_send_timeout ; 定义代理服务器向后端传输客户端请求的超时时长，两次连续的操作间隔时长，如果后端服务器在此时长内没有收到任何请求，则关闭连接；

            配置示例：
                proxy_cache webcache;
                proxy_cache_key $request_uri;
                proxy_cache_valid 200 302 10m;
                proxy_Cache_valid 301 1h;
                proxy_cache_valid any 1m;
                proxy_cache_methods GET HEAD;
    
    4. 修改响应报文首部配置
        ngx_http_headers_module

        代理服务器响应给客户端的响应报文添加自定义首部，或修改指定首部的值；

        add_header X-Via  $server_addr; $server_addr 代理服务器地址；$proxy_host proxied server地址；
        add_header X-Accel $server_name; $server_name 代理服务器主机名；

## Nginx做fastcgi反向代理服务器配置
    构建LNMP

   **ngx_http_fastcgi_module**

    1. fastcgi_pass address;
       设置fastcfi服务器地址，可以是FQDN或ip addr，同时给出端口号；
            fastcgi_pass FQDN:9000
            fastcgi_pass ip_addr:9000

       或者UNIX-domain socket文件:
            fastcgi_pass unix:/path/to/fastcgi.socket;

    2. fastcgi_index index.php;
        定义fastcgi代理的默认主页文件；

    3. fastcgi_param parameter value [if_not_empty];
        把客户端向nginx请求建立连接的状态参数存储于Nginx的变量，并将此变量转换为fastcgi可识别的变量名传递给fastcgi server；实现将连接状态信息传递给fastcgi server;

        fastcgi_param SCRIPT_FILENAME /PATH/TO/php$fastcgi_script_name

        /PATH/TO/：此处的/PATH/TO/指定在fastcgi server上请求uri的根目录；

        $fastcgi_script_name:请求的URI,如果请求的URI以"/"结束，追加fastcgi_index定义的index file至URI,类似于uri/index.php; 此变量可以用于SCRIPT_FILENAME和PATH_TRANSLATED 参数，传递请求的php程序文件名称；

        配置示例：
            location ~* \.php$ {
                fastcgi_pass 172.20.101.61:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME /appdata$fastcgi_script_name;
                include       fastcgi_params;
            }
        
        配置示例2：通过/pm_status和/ping来获取fpm server运行状态信息；
            location ~* ^/(pm_status|ping)$ {
                include fastcgi_params;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
            }

    4. fastcgi反向代理缓存配置

        定义缓存：fastcgi_cache_path ; 仅用于http上下文；
        调用缓存：fastcgi_cache

        (1)定义缓存：fastcgi_cache_path
            fastcgi_cache_path path [levels=level] keys_zone=name:size max_size= ;

            配置示例：
                fastcgi_cache_path /var/cache/fcgi levels=1:2 keys_zone=fcgicache:10m max_size=2G;

        (2)调用缓存：fastcgi_cache
            fastcgi_cache name;
            fastcgi_cache_key [$proxy_host]$request_uri；
            fastcgi_cache_valid 200 302 10m;
            fastcgi_cache_valid 301 1h;
            fastcgi_cache_valid any 1m;
            fastcgi_cache_methods GET HEAD;

            配置示例：
                fastcgi_cache_key localhost:9000$request_uri;

   