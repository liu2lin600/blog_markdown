---
title: keepalived+haproxy+varnish实现web动静分离
date: 2016-08-22 09:39:35
categories: Linux服务 
tags: [反向代理, 负载均衡, 缓存]
---

前端使用haproxy进行负载均衡及对动静分离，同时使用keepalived对其高可用，后端静态资源使用varnish进行缓存，以php为例对动态请求基于memcached保存session以实现会话保持效果

<!--more-->

按照惯例直接上本次实验的拓扑图，如下

![main](main.png)

## 环境准备
- 系统：CentOS.x86_64.7.2
- 为实验方便进行，关闭各主机防火墙及SELinux
- 所有程序使用yum安装，版本如下:
    - keepalived-1.2.13-7.el7.x86_64
    - haproxy-1.5.14-3.el7.x86_64 
    - varnish-4.0.3-3.el7.x86_64
    - httpd-2.4.6-40.el7.centos.4.x86_64
    - php-5.4.16-36.3.el7\_2.x86_64
    - memcached-1.4.15-9.el7.x86_64
    - php-pecl-memcached-2.2.0-1.el7.x86_64
 - 如上图在各主机上安装相应程序

_备注：正常情况下，haproxy主机应配置2块网卡，对外使用信公网IP，反代后端使用内网IP，本实验为方便统一使用同一网段_

## 后端动态服务配置
1. 修改httpd监听端口为8080
2. 为2台动态主机准备测试页面

    `► vim /var/www/html/index.php`
    ```php
        <?php
            session_start();

            $mem = new Memcached();
            $mem->addServer('172.16.60.74',11211);

            echo "backend_1".'<br/>';
            echo "SESSION_ID: ".session_id().'<br/>';
            $ss = $mem->get('memc.sess.key.'.session_id());
            print_r($ss);

            # 添加一个图片链接，测试动静分离
            echo "<br/><img src='./42.jpg' />";
    ```
3. 在172.16.60.73主机上添加一个设置session页面
    `► vim /var/www/html/session.php`
    ```php
        <?php
            $ip = '172.16.60.74';
            $port = 11211;
            ini_set("session.save_handler", "memcached");
            ini_set("session.save_path", $ip.":".$port."?persistent=1&weight=1&timeout=1&retry_interval=15");

            session_start();
            $_SESSION['test'] = 'kelly';
            if ($_SESSION['test']) echo 'set session success!';
    ```
4. 测试

## varnish配置
1. 修改varnish默认参数
    `vim /etc/varnish/varnish.params`
    ```bash
    # 修改为使用内存做缓存，注释默认项，其它保持不变
    VARNISH_STORAGE="malloc,200M"
    ```
2. 修改vcl配置文件
    `vim /etc/varnish/default.vcl` 
    ```bash
    vcl 4.0;

    # 健康检查
    probe check {
            .url = "/check.html";
            .window = 5;
            .threshold = 2;
            .interval = 10s;
    }

    acl purge {
            "127.0.0.1";
    }

    backend default {
        .host = "172.16.60.71";
        .port = "8080";
            .probe = check;
    }

    sub vcl_purge {
        return (synth(200,"Purged"));
    }

    sub vcl_recv {
        if (req.restarts == 0) {
            if (req.http.X-Forwarded-For) {
                set req.http.X-Forwarded-For = req.http.X-Forwarded-For + ", " + client.ip;
            } else {
                set req.http.X-Forwarded-For = client.ip;
            }
        }
        # 只允许通过本机对缓存做修剪
        if (req.method == "PURGE") {
            if (!client.ip ~ purge){
                return (synth(405, "This request is not allowed"));
            }
            return (purge);
        }
    }

    sub vcl_backend_response {
        # 公共图片等静态资源添加有效时长
        if (beresp.http.cache-control !~ "s-maxage") {
            if (bereq.url ~ "(?i)\.(jpg|jpeg|png|gif|css|js)$") {
                unset beresp.http.Set-Cookie;
                set beresp.ttl = 3600s;
            }
        }
    }

    sub vcl_deliver {
        # 添加响应首部记录是否缓存命中
        if (obj.hits > 0){
            set resp.http.X-Cache = "HIT From " + server.ip;
        } else {
            set resp.http.X-Cache = "MISS";
        }
    }
    ```
3. 172.16.60.71静态服务监听端口修改为8080，并添加测试页面
    在`/var/www/html/目录下添加一张图片(42.jpg)`
    ```bash
    # 添加主页
    echo "<h2>static page</h2>" > /var/www/html/index.html
    # 添加健康检测页
    echo "I'm alive" > /var/www/html/check.html
    ```
4. 启动varnish并载入vcl配置
    ![varnishadm](varnishadm.png)
    
5. 健康检测状态
    ![check](check.png)

6. 缓存修剪测试
    ![purge](purge.png)
    从上图可看出，varnish缓存只允许本机对其进行清除操作

## haproxy配置
1. 修改haproxy主配置文件，global及default段保持不变
    `vim /etc/haproxy/haproxy.cfg`
    ```bash
    # 访问入口
    frontend websrvs
        bind *:80
        acl  static     path_beg    -i  /static  /javascript  /stylesheets
        acl  static     path_end    -i  .html .js .css .jpeg .png .gif .bmp
        acl  dynamic    path_end    -i  .php

        use_backend     dynamic     if  dynamic
        use_backend     static      if  static

        default_backend static
        rspadd X-Via:\ HAPorxy

    # 静态资源使用cookie保持
    backend static
        cookie WEBSRV insert nocache indirect
        server srv1 172.16.60.71:6081 check weight 2 cookie srv1
        server srv2 172.16.60.72:6081 check weight 1 cookie srv2

    # 动态轮询
    backend dynamic
        balance roundrobin
        server  srv3 172.16.60.73:8080 check
        server  srv4 172.16.60.74:8080 check

    # 监控页面
    listen stats
        bind *:8009
        stats enable
        stats uri /admin?stats
        stats auth liu:lin
        stats realm admin\ area
        stats admin if TRUE
        stats hide-version
    ```

2. 启动haproxy并查看监控页
    ![stats](stats.png)

## keepalived配置
1. MASTER设置(172.16.60.3)
    `vim /etc/keepalived/keepalived.conf`
    ```bash
    !Configuration File for keepalived
    global_defs {
        notification_email {
            root@localhost
        }
        notification_email_from keepalived@localhost
        smtp_server 127.0.0.1
        smtp_connect_timeout 30
        router_id node1
        vrrp_mcast_group4 224.0.100.18
    }

    vrrp_script chk_haproxy {
        script "killall -0 haproxy"
        interval 1
        weight -5
    }

    vrrp_instance VI_1 {
        state MASTER
        interface eno16777736
        virtual_router_id 66
        priority 100
        advert_int 1

        authentication {
            auth_type PASS
            auth_pass sdfhj23H
        }

        virtual_ipaddress {
            172.16.60.100/32 dev eno16777736
        }
        track_script {
            chk_haproxy
        }
        notify_master "/etc/keepalived/notify.sh master"
        notify_backup "/etc/keepalived/notify.sh backup"
        notify_fault "/etc/keepalived/notify.sh fault"
    }
    ```
2. BACKUP配置(172.16.60.4)
    复制MASTER配置并做如下几处修改
    ```
    router_id node2
    state BACKUP
    priority 97
    ```

3. 准备通知脚本，用来当haparoxy不可用时发送邮件并尝试自动重启
    `vim /etc/keepalived/notify.sh` **并添加执行权限**
    ```
    #!/bin/bash
    contact='root@localhost'

    notify() {
        mailsubject="$(hostname) to be $1, vip floating."
        mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
        echo "$mailbody" | mail -s "$mailsubject" $contact
    }

    case $1 in
    master)
        notify master
        ;;
    backup)
        notify backup
        systemctl restart haproxy
        ;;
    fault)
        notify fault
        systemctl restart haproxy
        ;;
    *)
        echo "Usage: $(basename $0) {master|backup|fault}"
        exit 1
        ;;
    esac
    ```
4. 启动2台主机keepalived
    并查看vip(172.16.60.100)是否添加成功
    ![vip](vip.png)
    手动关闭主haproxy，查看备haproxy情况
    ![keepalived](keepalived.png)
    _主haproxy故障时，备启动，但主状态变成backup时触发通知脚本，重启haproxy，一旦重启成功并重新成为主节点_

## 缓存及会话保持测试
1. 缓存命中测试
    ![cache](cache.png)

2. 会话保持及动静分离
    ![session](session.png)
  
以上就是本次实验的全部内容