---
title: Nginx配置文件主要指令参考
date: 2016-08-02 16:55:18
categories: Linux服务
tags: nginx配置
---

对nginx核心模块及部分重要模块指令进行说明
<!--more-->

详情参考官网：[nginx官方文档](http://nginx.org/en/docs/)

## 配置文件组成
配置由指令和值组成，每个指令以';'结尾，主要的上下文包括main, events, http, upstream, server, location，结构如下

```bash
main BLOCK      # 主配置段
events {        # 事件相关配置
    ...
}
http {          # http/https相关
    ...
    upstream {  # 负载均衡服务器配置
        ...
    }
    server {    # 虚拟主机
        ...
        location /xxx {     # URI定位后配置
            ...
        }
    }
}
stream {        # tcp代理段
    ...
}
mail {          # 邮件代理配置
    ...
}
```

## 核心模块配置段
1. 正常运行的必备配置
    - `user username [groupname];`     
        指定运行worker进程的用户和组
    - `pid /path/to/pidfile_name; `    
        指定nginx的pid文件
    - `worker_rlimit_nofile 10240;`    
        指定一个worker进程所能够打开的最大文件句柄数
    - `worker_rlimit_sigpending N;`    
        设定每个用户能够发往worker进程的信号的数量

2. 优化性能相关的配置
    - `worker_processes NUM|auto;`        
        worker进程的个数，通常其数值应该为CPU的物理核心数减1
    - `worker_cpu_affinity cpumask ...;`  
        绑定cpu亲缘性
        ```
        0000 0001 == 第一颗
        0000 0010 == 第二颗
        ```
    - `worker_priority NICE;`             
        NICE值在-20~19之间，越低优先级越高
    - `ssl_engine device;`                
        在存在ssl硬件加速器的服务器上，指定所使用的ssl硬件加速设备

3. 事件相关的配置
    - `worker_connections 2048;`       
        每个worker能够并发响应最大请求数
    - `use epoll;`                     
        定义使用的事件模型，建议让nginx自动选择
    - `accept_mutex [on|off];`
        是否打开Ningx的负载均衡锁，此锁能够让多个worker进轮流地、序列化地与新的客户端建立连接，而通常当一个w    orker进程的负载达到其上限的7/8，master就尽可能不再将请求调度此worker。off时一个连接请求唤醒多个worke
    r 进程，同时只有一个worker获得连接，存在惊群问题，不过由于nginx进程数少，问题不大
    - `accept_mutex_delay NUMms;`
        accept锁模式中，一个worker进程为取得accept锁的等待时长，如果某worker进程在某次试图取得锁时失败了，至    少要等待#ms才能再一次请求锁
    - `lock_file /path/to/lock_file;`  
        lock文件
    - `multi_accept on|off;`           
        是否允许一次性地响应多个用户请求，默认为Off 

4. 用于调试及定位问题配置
    - `daemon on|off;`          
        ningx运行后台默认为on，调试时可设为off，所有信息接输出控制台
    - `master_process on|off;`
        是否以master/worker模式运行，默认为on，调试时可设off以方便追踪
    - `error_log /path/to/error_log LEVEL;`
        错误日志文件及其级别，默认为error级别，调试时可以使用debug级别，但要求在编译时必须使用--with-debug启用debug功能


## http模块配置段
1. 虚拟主机相关的配置 
    - `server {...}`
        定义一个虚拟主机,nginx支持使用基于主机名或IP的虚拟主机
            server {
                listen 80;
                server_name www.liu2lin.com;
                root /web/www;
            }  

    - `listen address[:port];` 
        `listen port;`  
        `listen unix:path;`  
            **其它可用参数：**
            default_server：默认虚拟主机，定义此server为http中默认的server，如果所有的server中没有任何一个listen使用此参数，那么第一个server即为默认server;
            ssl：限制只能通过ssl连接提供服务
            back
            rcvbuf=SIZE：接收缓冲大小
            sndbuf=SIZE：发送缓冲大小

    - `server_name [...];`
        当前server的主机名，可指定一个或空白字符分隔的多个主机，支持使用通配及正则匹配，优先级如下
            1. 首先做精确匹配；www.liu2lin.com 
            2. 左侧通配符匹配；*.liu2lin.com
            3. 右侧通配符匹配；www.abc.com, www.*
            4. 正则表达式匹配: ~^.*\.liu2lin\.com$

    - `server_name_hash_bucket_size 32|64|128;`
        为了实现快速主机查找，nginx使用hash表来保存主机名

2. 路径定义相关配置
    - `root PATH;`
        设置web资源路径，用于指定请求的根文档目录，可用于http, server, location中

    - `location [ = | ~ | ~* | ^~ ] uri { ... }`
        `location @name { ... }`
        功能：允许根据用户请求的URI来匹配指定的各location以进行访问配置；匹配到时，将被location块中的配置所处理

        `=`： URI精确匹配
        `~`： 正则表达式模式匹配，匹配时区分字符大小写
        `~*`：正则表达式模式匹配，匹配时忽略字符大小写
        `^~`：URI前半部分匹配，不区分字符大小写

            location ~* .*\.(jpg|jpeg|gif|png)$ {
                root /web/images;
            }

        匹配优先级：= > ^~ > ~|~* > 无符号  
        字符字面量最精确匹配、正则表达式检索（由第一个匹配到所处理）、按字符字面量

    - `alias PATH;`  
        用于定义的路径别名，只能用于location中
            location /data/ {
                alias /web/images/; 
            }
                
            # http://www.a.com/data/a.jpg ==> http://www.a.com/web/images/a.jpg

        注：
        root指令：给定的路径对应于location中的URI左侧的'/'
        alias指令：给定的路径对应于location中的URI右侧的'/'

    - `index file ...;`
        定义默认页面，可跟多个值

    - `error_page code ... [=[response]] uri;`
        自定义错误页重定向，当对于某个请求返回错误时，如果匹配上了error_page指令中设定的code，则重定向到新的URI中，默认返回相应响应码，如加上'=xxx'，则发生相应改变
            error_page  404  /404.html
            error_page  500  502  503  /50x.html

    - `try_files path1 [path2 ...] uri;`
        自左至右尝试读取由path所指定路径，在第一次找到即停止并返回，如果所有path均不存在，则返回最后一个uri; 
            location ~* ^/documents/(.*)$ {
                root /www/htdocs;
                try_files $uri /docu/$1 /temp.html;
            }

            # http://www.liu2lin.com/documents/a.html
            # http://www.liu2lin.com/docu/a.html
            # http://www.liu2lin.com/temp.html

3. 网络连接相关的设置：
    - `keepalive_timeout TIME;`
        保持连接的超时时长，默认为75秒，0表示禁止使用长连接

    - `keepalive_requests N;`
        在一次长连接上允许承载的最大请求数，默认100个

    - `keepalive_disable [msie6 | safari | none ];`
        对指定的浏览器禁止使用长连接

    - `tcp_nodelay on|off`
        对keepalive连接是否使用TCP_NODELAY选项，DELAY为将多个报文合并再一起发送

    - `client_header_timeout TIME;` 
        读取http请求首部的超时时长，默认60s

    - `client_body_timeout TIME;`
        读取http请求包体的超时时长，默认60s

    - `send_timeout TIME;`
        向客户端发送响应的超时时长，指两次写操作之间的间隔时长

4. 对客户端请求的限制：
    - `limit_except METHOD ... { ... };`
        指定对范围之外的其它方法的访问控制，只能用于location中
            # 表示除了GET以外的方法仅允许172.16网络主机访问
            limit_except GET {
                allow 172.16.0.0/16;
                deny all; 
            }

    - `client_max_body_size SIZE;`
        http请求包体的最大值，常用于限定客户所能够请求的最大包体，根据请求首部中的Content-Length来检测，以避免无用的传输

    - `limit_rate RATE;`
        限制客户端每秒钟传输的字节数，默认为0，表示没有限制
            location /download/ {
                root /web/;
                limit_rate 20480;
            }

    - `limit_rate_after TIME;`
        nginx向客户发送响应报文时，如果时长超出了此处指定的时长，则后续的发送过程开始限速

5. 文件操作的优化：
    - `sendfile on|off;`
        是否启用sendfile功能，开启后会减少用户空间到内核空间的上下文切换。对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，默认off

    - `aio on|off;`
        是否启用aio(异步非阻塞)功能

    - `open_file_cache max=N [inactive=TIME]|off;`
        是否打开文件缓存功能

        `max`：缓存条目的最大值，当满了以后将根据LRU算法进行置换
        `inactive`：缓存条目在指定时长时没有被访问过时，将自动被删除，默认为60s

        缓存的信息包括：
        1. 文件句柄、文件大小和上次修改时间
        2. 已经打开的目录结构
        3. 没有找到或没有访问权限的信息

    - `open_file_cache_errors on|off;`
        是否缓存文件找不到或没有权限访问等相关信息

    - `open_file_cache_valid TIME;`
        多长时间检查一次缓存中的条目是否超出非活动时长，默认为60s

    - `open_file_cache_min_use N;`
        在inactive指定的时长内被访问超此处指定的次数地，才不会被删除

6. 对客户端请求的特殊处理：
    - `ignore_invalid_headers on|off;`
        是否忽略不合法的http首部，默认为on，off意味着请求首部中出现不合规的首部将拒绝响应。只能用于server和http

    - `log_not_found on|off;`
        是否将文件找不到的信息也记录进错误日志中

    - `resolver ADDRESS;`
        指定nginx使用的DNS服务器地址

    - `resover_timeout TIME;`
        指定DNS解析超时时长，默认为30s

    - `server_tokens on|off;`
        是否在错误页面中显示nginx的版本号，建议关闭

7. 内存及磁盘资源分配：
    - `client_body_in_file_only on|clean|off;`
        HTTP的包体是否存储在磁盘文件中；非off表示存储，即使包体大小为0也会创建一个磁盘文件；on表示请求结束后包体文件不会被删除，clean表示会被删除

    - `client_body_in_single_buffer on|off;`
        HTTP的包体是否存储在内存buffer当中；默认为off

    - `cleint_body_buffer_size size;`
        接收请求包体的内存缓冲区大小，默认16k，超出时将被暂存在磁盘上

    - `client_body_temp_path PATH [level1 [level2 [level3]]];`
        HTTP包体存放的临时目录，对应几位16位进制个数
            client_body_temp_path /var/tmp/client/  1 2 ==> /var/tmp/client/{0-f}/{00-ff}

    - `client_header_buffer_size size;`
        正常情况下接收用户请求的http报文header部分时分配的buffer大小，默认为1k

    - `large_client_header_buffers number size; `
        存储超大Http请求首部的内存buffer大小及个数

    - `connection_pool_size size;`
        nginx对于每个建立成功的tcp连接都会预先分配一个内存池，此处即用于设定此内存池的初始大小；默认为256

    - `request_pool_size size;`
        nginx在处理每个http请求时会预先分配一个内存池，此处即用于设定此内存池的初始大小；默认为4k


## 核心模块常用内置变量

`$uri`：当前请求的uri，不带参数
`$request_uri`：请求的uri，带完整参数
`$host`：http请求报文中host首部，如果请求中没有host首部，则以处理此请求的虚拟主机的主机名代替
`$hostname`：nginx服务运行在的主机的主机名
`$remote_addr`：客户端IP
`$remote_port`：客户端Port
`$remote_user`：使用用户认证时客户端用户输入的用户名
`$request_filename`：用户请求中的URI经过本地root或alias转换后映射的本地的文件路径
`$request_method`：请求方法
`$server_addr`：服务器地址
`$server_name`：服务器名称
`$server_port`：服务器端口
`$server_protocol`：服务器向客户端发送响应时的协议，如http/1.1, http/1.0
`$scheme`：在请求中使用scheme, 如https://www.magedu.com/中的https
`$http_HEADER`：匹配请求报文中指定的HEADER，如$http_host匹配请求报文中的host首部
`$sent_http_HEADER`：匹配响应报文中指定的HEADER，如$http_content_type匹配响应报文中的content-type首部
`$document_root`：当前请求映射到的root配置

## 部分重要模块

### ngx_http_headers_module：响应报文首部设置
- `add_header NAME VALUE [always];`
- `expires [modified] time ｜ epoch | max | off;`
        expires    24h;
        expires    modified +24h;
        expires    @24h;
        add_header X-Via $server_addr;              # 添加代理主机地址
        add_header X-Cache $upstream_cache_status;  # 添加代理缓存使用状况，有HIT,MISS,BYPASS...
        add_header Cache-Control private;

### ngx_http_access_module：实现基于ip的访问控制功能
- `allow address | CIDR | unix: | all;`
- `deny address | CIDR | unix: | all;`
        location / {
            deny  192.168.1.1;
            allow 192.168.1.0/24;
            allow 10.1.1.0/16;
            allow 2001:0db8::/32;
            deny  all;
        }

### ngx_http_auth_basic_module：用户认证
- auth_basic STRING | off;    # 使用basic机制进行用户认证或关闭
- auth_basic_user_file FILE;  # 认证用的账号密码文件

        location /admin/ {
            root /www/b.org;
            auth_basic "admin area"|off;        # 提示信息或关闭认证
            auth_basic_user_file /etc/nginx/.htpasswd;
        }
        
        # 密码文件格式：USER:PASS:COMMENT
        htpasswd -c -m /etc/nginx/.htpasswd tom     # 生成密码文件，第一次生成使用'-c'

### ngx_http_autoindex_module：文件索引
    # 显示/www/b.org/download/下的各资源列表
    location /download/ {
        root /www/b.org;
        autoindex on; 
    }

### ngx_http_stub_status_module：nginx状态显示
    location /stub_status {
        stub_status;        # 开启状态显示
        allow 192.168.0.0/16;
        deny all;
    }
            
访问显示内容分析：
    
    Active connections: 291 
    server accepts handled requests
         16630948 16630948 31070465 
    Reading: 6 Writing: 179 Waiting: 106

    # active connections:对后端发起的活动连接数
    # accept:已经接受的16630948个连接
    # handled:成功创建16630948次握手(与accept一样说明没出现失败)
    # requests:总共处理了31070465个请求(1.8个/s)
    # Reading: nginx读取到客户端的Header信息数 
    # Writing: 返回给客户端的Header信息数 
    # Waiting: 开启 keep-alive 的情况下，这个值等于前两者之和

### ngx_http_referer_module：防盗链
- `valid_referers none|blocked|server_names|string ...;`   # 定义合规的引用
    `none`：无referer，即直接输入网址
    `blocked`：判断合法性

        location ~* \.(jpg|png|gif|jpeg)$ {
            root /web/liu2lin;
            valid_referers none blocked www.liu2lin.com;          # 定义
            if($invalid_referer){                                 # 判断引用是否符合
                rewrite ^/.*$ http://www.liu2lin.com/403.html     # 不符合重写到403页面
            }
        }

### ngx_http_rewrite_module：URL重写
- `rewrite REGEX REPLACE [flag];`

[flag]标志：
`last`：一旦被当前规则匹配并重写后立即停止检查后续的其它rewrite的规则，而后通过重写后的规则重新发起请求
`break`：一旦被当前规则匹配并重写后立即停止后续的其它rewrite的规则，而后继续由nginx进行后续操作
`redirect`：不是以http://或https://开头，返回302临时重定向
`permanent`：返回301永久重定向

    location / {
        root /www/b.org;
        rewrite ^/images/(.*)$ /imgs/$1 last; 
        rewirte ^/imgs/(.*)$ /images/$1;
    }
    # http://www.b.org/images/a.jpg --> http://www.b.org/imgs/a.jpg

    location /download/ {
        rewrite ^(/download/.*)/media/(.*)\..*$ $1/media/$2.mp3 break;
    }

    # 注意：nginx最多循环10次，超出之后会返回500错误，一般将rewrite写在location中时都使用break标志，或者将rewrite写在if上下文中

- `rewrite_log on|off;`
    是否把重写过程记录在错误日志中，默认为notice级别off

- `return CODE;`
    用于结束rewrite规则，并且为客户返回状态码，可以使用的状态码有204, 400, 402-406, 500-504等

- `set $var VALUE;` # 设置自定义变量

- `if (condition) { ... }`
    _condition比较操作符：_
    `==`, `!=`, `!~`, `!~*`
    `~`：模式匹配，区分字符大小写
    `~*`: 模式匹配，不区分字符大小写

    _文件及目录存在性判断：_
    `-e`, `!-e`：存在与否
    `-f`, `!-f`：存在且为一个普通文件与否
    `-d`, `!-d`：目录
    `-x`, `!-x`：执行

### ngx_http_gzip_module：压缩，适用于http, server, location
- `gzip on | off;`
    开启或关闭压缩
- `gzip_buffers 32 4k;`         
    用于实现压缩功能的缓存区数量及大小
- `gzip_comp_level 6;`          
    压缩级别1-9，默认1
- `gzip_disable regex ...;`     
    对客户端浏览器类型字符串匹配至此处的regex所描述的模式的请求禁用压缩功能
- `gzip_min_length LENGTH;`     
    默认20字节
- `gzip_proxied off |expired|no-cache |no-store|private|no_last_modified|no_etag|auth|any ...;`
    对代理的请求所获取的响应报文是否启动压缩功能，以及如何启用，默认关闭
- `gzip_types mime-type ...; ` 
    压缩过滤器，仅对此处设定的类型的内容启用压缩功能，默认为text/html

### ngx_http_fastcgi_module：fastcgi代理，nginx与php结合只能通过fastcgi协议
- `fastcgi_pass address;`         
    address为php-fpm服务器监听的地址
- `fastcgi_index name; `          
    fastcgi默认的主页资源
- `fastcgi_param  parameter value [if_not_empty];`      
- `fastcgi_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];`       
    缓存设置，定义在http中 
        path                    # 缓存目录路径；
        levels=                 # 缓存目录层级数量，以及每一级的目录数量
            levels=1:2:1        # 表示一级目录16个，每个1级目录下256个，第个2级目录下16个
        keys_zone=NAME:SIZE     # 内存中的缓存空间的名称及大小
        max_size=SIZE           # 磁盘上用于缓存数据的缓存空间上限
        inactive=TIME           # 缓存时长
                
- `fastcgi_cache zone | off;`         
    调用指定的缓存空间来缓存数据
- `fastcgi_cache_key STRING;`         
    定义用作缓存项的key的字符串
- `fastcgi_cache_methods GET...;`     
    对哪些请求方法来检查缓存
- `fastcgi_cache_min_uses NUM;`       
    缓存有指定时间内最少引用
- `fastcgi_cache_valid [code ...] time;`      
    对不同的响应码的资源可缓存时长
                
    注意：调用缓存时，至少应该设定三个参数
    `fastcgi_cache`
    `fastcgi_cache_key`
    `fastcgi_cache_valid` 
            
        http{
            fastcgi_cache_path  /var/nginx/cache  key_zone=WORD:10m level=1:2 
            
            location / {
                fastcgi_pass  127.0.0.1:9000;
                fastcgi_index index.php;

                fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name;
                include       fastcgi_params;

                fastcgi_cache  WORD;
                fastcgi_cache_key  $request_uri;
                fastcgi_cache_valid 200 302 1h;
                fastcgi_cache_min_uses  2;
            }
        }

### ngx_http_log_module：日志记录模块
- `access_log PATH [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]]|off;`
    日志路径、格式、缓冲等设置，默认access_log logs/access.log combined;  combined格式无需定义
                
- `log_format NAME string ...;`     
    日志格式设置
                
    内置combined格式及反代格式时：
        log_format combined '$remote_addr - $remote_user [$time_local] '
                            '"$request" $status $body_bytes_sent '
                            '"$http_referer" "$http_user_agent"';
        log_format  porxy   '$http_x_forwarded_for - $remote_user [$time_local] '
                            '"$request"  $status $body_bytes_sent '
                            '"$http_referer"  "$http_user_agent" ';

- `open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time];`
    日志文件缓存，默认是off
        max:设置缓存中的最大文件描述符数量，如果被占满，采用LRU算法将描述符关闭
        inactive:设置存活时间，默认是10s
        min_uses:在inactive时间段内，缓存最少需被使用次数，默认1
        valid:设置检查频率，默认60s

### ngx_http_proxy_module：反向代理
接收完整报文分析后先查看本地缓存是否有需要资源，如果没有再向后端转发(将自己当作客户端去请求资源)

- `proxy_pass URL; `
    用于location, if in location, limit_except
                
    注意：
    (1) proxy_pass后面的路径不带uri时，其会将location的uri传递给后端主机
        location /uri/ {
            proxy_pass http://HOST;
        }
            http://xx/img/a.jpg ==> http://HOST/img/a.jpg
        
    (2) proxy_pass后面的路径是一个uri时，其会将location的uri替换为proxy_pass的uri
        location /uri/ {
            proxy_pass http://HOST/new_uri/;
        }
            http://xx/uri/a.jpg ==> http://HOST/new_uri/a.jpg
        
    (3) location定义uri使用正则表达式，则proxy_pass之后必须不能使用uri
        location ~|~* PATTERN {
            proxy_pass http://HOST;
        }
                    
- `proxy_set_header field value;`
    设定发往后端主机的请求报文的请求首部的值，后端访问日志做相应修改
    代理设置：proxy_set_header X-Real-IP $remote_addr;
    后端主机日志修改：
        LogFormat "%{X-Real-IP}i %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined

- `proxy_cache_path; ` 
    定义代理缓存，只能用于http中
            
        proxy_cache_path PATH [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];
                    
- `proxy_cache zone | off;`          
    开启或禁用缓存
- `proxy_cache_key string;`          
    缓存条目的键
- `proxy_cache_valid [code] time;`   
    对各类响应码的缓存时长
**注：正常以上3项内容必备**
            
- `proxy_cache_use_stale error|timeout|invalid_header|updating|http_code|off ...;`
- `proxy_connect_timeout TIME`          
    `proxy_read_timeout TIME`
    `proxy_send_timeout TIME`
    代理连接、请求、响应超时时长
                
- `proxy_buffering on|off`  
    `proxy_buffer_size SIZE`
    `proxy_buffers NUM SIZE`
    开启代理缓冲、缓冲数量及大小

        # 先定义缓存设置，只能定义在http中
        http{
            proxy_cache_path /var/cache/nginx/proxy_cache levels=1:2:1 keys_zone=pcache:10m max_size=1g;
        }
            
        server {
            proxy_set_header X-Real-IP $remote_addr;    # 添加客户端IP给后端服务器
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for     # 多级代理追加

            proxy_cache pcache;             # 开启缓存
            proxy_cache_key $request_uri;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 301 1h;
            proxy_cache_valid any 1m;  

            location / {
                proxy_pass http://192.168.1.8;
            }
        }

### ngx_http_upstream_module：设置负载均衡

- `ip_hash;`    
    实现IP绑定
- `server ADDR [weight=N] [max_fails=N] [fail_timeout=TIME] [slow_start=TIME] [backup] [down]...;`
        ADDR格式：
            IP[:PORT]
            HOSTNAME[:PORT]
            unix:/PATH/TO/SOME_SOCK_FILE
        weight：权重
        max_fails：最大失败连接次数，失败连接的超时时长由fail_timeout指定
        fail_timeout：等待请求的目标服务器发送响应的时长
        backup：用于fallback的目的，所有服务均故障时才启动此服务器
        down：手动标记其不再处理任何请求

- `least_conn;`
    最少连接调度算法； 当server拥有不同的权重时为wlc
- `hash key [consistent];`
    基于指定的key的hash表实现请求调度，此处的key可以文本、变量或二者的组合
    consistent：参数，指定使用一致性hash算法
    
        hash $request_uri consistent
        hash $remote_addr
        hash $cookie_name
                        
- `keepalive NUM;`      
    可使用长连接的连接数量，默认关闭


**1. upstream只能用于http上下文**
**2. 各server只能直接使用IP或主机名，不要加协议**

        upstream dynamic {
            zone upstream_dynamic 64k;

            server backend1.liu2lin.com      weight=5;
            server backend2.liu2lin.com:8080 fail_timeout=5s slow_start=30s;
            server 192.0.2.1                 max_fails=3;
            server backend4.liu2lin.com      service=http resolve;

            server backup.liu2lin.com:8080  backup;
        }

        server {
            location / {
                proxy_pass http://dynamic;
            }
        }

### ngx_stream_core_module：设置tcp/udp代理
        
- listen address:port [ssl] [udp] [backlog=number] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
    默认监听tcp协议
        # 代理ssh      
        stream {
            upstream sshsrvs {
                server 192.168.1.8:22;
                server 192.168.1.9:22;
                hash $remote_addr consistent;
            }

            server {
                listen 172.16.60.4:22022;
                proxy_pass sshsrvs; 
            }
        }

_以上为常用部分说明，未完待续..._