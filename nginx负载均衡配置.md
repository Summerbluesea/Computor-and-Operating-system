## Nginx实现七层代理和负载均衡的配置

    Nginx利用ngx_http_upstream_module定义server组和调度算法，定义的server组可被proxy_pass, fastcgi_pass, memcached_pass引用从而完成负载均衡调度；

   **ngx_http_upstream_module**

   The ngx_http_upstream_module module is used to define groups of servers that can be referenced by the proxy_pass, fastcgi_pass, uwsgi_pass, scgi_pass, memcached_pass, and grpc_pass directives.

    1. upstream
        定义server组，组内servers可以监听不同的端口，也可以混用监听于UNIX-domain套接字；仅用于http上下文；

        upstream模块默认调度算法round-robin，可以使用weight定义组内server的权重；

        http {
            upstream staticweb {
                server 172.17.0.2 weight=2;
                server 172.17.0.3;
            }

            location / {
                proxy_pass http://staticwebsrvs;
            }
        }

    2. 调度算法之least_conn
        最少连接，其默认就考虑服务器权重，定义于upstream上下文；

        upstream staticweb {
            least_conn;
            server 172.17.0.2 weight=2;
            server 172.17.0.3;
        }

        长连接时使用least_conn调度算法，短链接使用round-robin；

    3. 调度算法之ip_hash
        客户端ip地址hash值对组内服务器权重之和取模，得到的数值即为调度至的upstream server；实现会话保持；

        存在缺点：此算法使用静态hash(static hashing)，也就是说客户都拿的IP地址是对可能发生变化的值取模，从而决定调度目标；如果组内一个服务器宕机，会影响整个server group的调度；

            static hashing: 静态hash；对一个可能发生变化的值(server group中服务器权重之和)取模；如果其中一台服务器宕机，会影响整个server group调度；

            consistent hashing:对一个固定值取模；一个数值对2^32取模得到的数值为 0 - 2^32-1, 把这2^32个每个整数看作一个刻度分布在一个环上，称为hash环；

                每一台服务器按权重换算为相当个数的虚拟服务器，虚拟服务器个数等于服务器权重之和；对每一个虚拟主机ip+salt做hash计算，再用此hash值对2^32取模，则每台虚拟服务器落在hash环的某个刻度上；

                当对客户端请求进行调度时，用客户端ip对2^32取模，得到的值也应该落在hash环上的某个刻度上，顺时针延环找到的第一台虚拟服务器即是调度至的服务器；如果调度找到的第一台服务器宕机，还可以顺时针继续找第二台服务器，不会影响整个server group的调度和服务，此即为一致性hash；

                为了避免hash环偏斜，即服务器映射至hash环位置集中在hash环某一范围；引入虚拟节点，类似于把服务器权重成比例放大，从而增加虚拟主机个数，而且，每个服务器服务的区域离散为hash环上的小范围，如果一台服务器宕机后，影响范围是多个离散的小局部；

            upstream staticweb {
            ip_hash;
            server 172.17.0.2 weight=2;
            server 172.17.0.3;
            }

    4. 调度算法之hash 
        客户端请求报文中的任何内容都可以做为hash key，做为调度参考标准；实现会话保持；

        hsah key [consistent]; 用于upstream上下文；

            key：定义用请求报文的什么内容做hash计算；
            consistent:选用一致性hash算法；$remote_addr , $request_uri;

        客户端ip地址做hash计算，将同一ip的请求调度至同一个服务器上；
        upstream staticweb {
            hash $remote_addr consistent;
            server 172.17.0.2 ;
            server 172.17.0.3;
            }

        客户端请求的URI做hash计算，将对相同URI请求调度至同一个服务器上，如果后端服务器是某反向代理的缓存服务器，利用此方法可以极大提高缓存命中率；
        upstream staticweb {
            hash $request_uri consistent;
            server 172.17.0.2 ;
            server 172.17.0.3;
            }

    5. 调度算法sticky
        前面介绍的ip_hash和hash算法可以从不同角度实现会话保持；sticky调度算法有三种方法可以(cookie,route,learn)实现session保持;下面主要介绍基于cookie的session保持；

        sticky cookie name [expires=time] [domain=domain] [httponly] [secure] [path=path];

            name:设定cookie名称；cookie的值是客户端ip+port或UNIX-domain docket的MD5 hash值的十六进制表示；
            expires=time:设定客户端浏览器得到的cookie有效时长，max表示将在 31 Dec 2037 23:55:55 GMT 过期；默认在当前浏览器会话结束时失效；
            domain:定义cookie生效的域；
            httponly:为cookie添加HttpOnly属性；
            path=path:定义cookie的路径；

        当使用sticky cookie时，调度指定的服务器信息将在由nginx生成的http cookie传递；

        upstream staticweb {
            sticky cookie web_id expires=1h domain=.itunes.top path=/;
            server 172.17.0.2 ;
            server 172.17.0.3;
            }

        客户端的第一次请求还没有绑定于特定的服务器上，客户端请求按配置的调度算法选择出的upstream server上；持有此cookie的客户端以后的请求会被传递给固定的服务器；如果指定服务器不能处理请求，如果客户端还没有与某个server绑定，nginx会选择一个新server处理请求；

    6. upstream模块的server配置
        可定义参数：
        weight=number; 定义server权重；
        max_conns=number; 定义server的最大并发连接数；

        可以定义做server健康状态检查：

        max_fails=number; 设置fail_timeout时长内与upstream server未建立连接的尝试次数，超过此设定值则认为server不可用；
        fail_timeout=time; 设置fail_timeout超时时长，超过此时间不能建立连接则认为server不可用；默认为10s;

        backup:标记server为备用；当主服务器宕机后，请求由backup server处理；

        down：标记server永久不可用；

        upstream staticweb {
                server 172.17.0.2 weight=2 fail_timeout=2 max_fails=2;
                server 172.17.0.3; fail_timeout=2 max_fails=2
            }


    7. keepalive配置
        nginx与upstream server保持连接数；可以提升响应性能，并且减少nginx客户端套接字占用数量；

        The connections parameter sets the maximum number of idle keepalive connections to upstream servers that are preserved in the cache of each worker process. When this number is exceeded, the least recently used connections are closed.

        keepalive connections; 


        实验：使用docker运行2个nginx做uptream server, 宿主机做负载均衡；使用默认调度算法round-robin;
        1. 运行两个docker 
        # docker run --name web1 -d -v /vols/web1/:/usr/share/nginx/html/ nginx:1.15-alpine
        # docker run --name web2 -d -v /vols/web2/:/usr/share/nginx/html/ nginx:1.15-alpine

        2. 配置调度器
        定义upstream 
        http {
            upstream staticweb {
                server 172.17.0.2;
                server 172.17.0.3;
                }

                server {
            listen       80 default_server;
            listen       [::]:80 default_server;
            server_name  www.itunes.top;

            location / {
            proxy_pass http://staticweb;
            }
        }

        3. 测试负载均衡调度
        [root@mysql_slave ~]#while true; do curl 172.18.134.134; sleep 1s;done
            <h1>web1 server</h1>
            <h1>web2 server</h1>
            <h1>web1 server</h1>
            <h1>web2 server</h1>
            <h1>web2 server</h1>
            <h1>web1 server</h1>
            <h1>web2 server</h1>

## Nginx实现四层代理和负载均衡配置

    Nginx在用户空间代为监听某一个端口接收用户请求，但自己不对应用层内容做任何解析，仅在nginx与被代理的后端服务器建立通道，根据客户端请求ip和端口，将客户端请求发给后端server；类似于DNAT,支持端口映射；

    如果后端提供的服务器有多台，nginx还可以基于uptream模块将后端服务器分成不同server group，并指定调度算法，从而面向面向客户端的nginx可以将客户端请求调度至特定server上；


    1. 四层代理配置

       基于ngx_stream_proxy_module实现；stream与http不同时使用；

        proxy_pass $upstream;

    2. 四层负载均衡

       基于ngx_stream_upstream_module实现；
       The ngx_stream_upstream_module module (1.9.0) is used to define groups of servers that can be referenced by the proxy_pass directive.

       支持的调度算法有hash, least_conn, least_time,random, 默认调度算法为round-robin；

       upstream name {...}; stream上下文；

        配置示例：

            stream {
                upstream backend {
                    hash $remote_addr consistent;

                    server backend1.example.com:12345 weight=5;
                    server 127.0.0.1:12345            max_fails=3 fail_timeout=30s;
                    server unix:/tmp/backend3;
                }

                server {
                    listen 12345;
                    proxy_connect_timeout 1s;
                    proxy_timeout 3s;
                    proxy_pass backend;
                }

                server {
                listen [::1]:12345;
                proxy_pass unix:/tmp/stream.socket;
                }
            }

