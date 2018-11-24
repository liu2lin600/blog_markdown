---
title: LAMP搭建的几个方案
date: 2016-07-16 10:03:59
categories: Linux服务
tags: [lamp, xcache]
---

在虚拟机中分别使用yum及源码包编译搭建lamp环境，并做测试

<!-- more -->


## 方案

1. CentOS 7, lamp+xcache, rpm包, php以module方式安装  
    a) 一个虚拟主机提供phpMyAdmin，另一个虚拟主机提供wordpress  
    b) 为phpMyAdmim提供https服务  

2. CentOS 7, lamp + xcache， rpm包，php-fpm  
    a) httpd, php, mariadb分别部署在一个单独的主机上  
    b) 一个虚拟主机提供phpMyAdmin，另一个虚拟主机提供wordpress  
    c) 为phpMyAdmim提供https服务

3. CentOS 7, lamp + xcache，编译安装，php-fpm  
    a) httpd, php, mariadb分别部署在一个单独的主机上，以及都在同一主机  
    b) 一个虚拟主机提供phpMyAdmin，另一个虚拟主机提供wordpress  
    c) 为phpMyAdmim提供https服务  

## 环境准备

- 系统：CentOS7.x86_64 (192.168.1.71~74)
- 配置好yum源并安装开发包组 **"Development Tools"**及**"Server Platform Development"**
- 为方便测试关闭防火墙iptables及SELinux，相关知识后期讨论
- 下载相应源码包：
    + apr-1.5.2.tar.bz2
    + apr-util-1.5.4.tar.bz2
    + httpd-2.4.23.tar.bz2
    + mariadb-5.5.50-linux-x86_64.tar.gz
    + php-5.5.37.tar.xz
    + xcache-3.2.0.tar.bz2
    + wordpress-4.5.3-zh_CN.tar.gz
    + phpMyAdmin-4.6.3-all-languages.zip

## 一、yum快速安装1，php以模块方式安装
### 1. 环境：静态IP(192.168.1.71)

### 2. 安装

```
yum -y install httpd php php-mysql mariadb-server php-xcache
```

### 3. httpd-2.4

- **配置文件：**/etc/httpd/conf/httpd.conf, /etc/httpd/conf.d/*.conf
- **启动服务：**  
    `systemctl start httpd`  
    `ss -tunl sport = :80`  
    `ps aux | grep httpd`  
- **测试：**  
    `echo "<h2>hello world</h2>" > /var/www/html/index.html`
    `curl 192.168.1.71`

### 4. php-5.4
- **配置文件：**/etc/php.ini, /etc/php.d/*.ini

### 5. mariadb-5.5
- **配置文件：**/etc/my.cnf,/etc/my.cnf.d/*.cnf  
    在主配置文件/etc/my.cnf中[mysqld]下添加：  
    `innodb_file_per_table = On`  
    `skip_name_resolve = On`

- **启动服务：**  
    `systemctl start mariadb`  
    `ss -tunl sport = :3306`  
    `ps aux | grep mysql`   
    *注：如果启动不了，尝试修改权限:* `chown -R mysql.mysql /var/lib/mysql`

- **初始化数据库及安全设置：**默认只允许root用户本地登录，且空密码  
    `mysql_install_db`  
    `mysql_secure_installation`  # 添加root密码，移除匿名用户...  

- **登录并授权：**  
    `mysql -uroot -p123`  
    `GRANT ALL ON *.* TO test@'localhost' identified by '123'`  
    `GRANT ALL ON *.* TO test@'127.0.0.1' identified by '123'`

- **使用测试用户登录：**  
    `mysql -utest -p123`

### 6. php-xcache
- **配置文件：**/etc/php.d/xcache.ini

### 7. 测试连接

```bash
vim /var/www/html/index.php     # 编辑默认主页内容如下
curl 192.168.1.71/index.php     # 测试是否连接成功
```
```php
<?php
    $con = mysql_connect('127.0.0.1','test','123');
    if($con){
        echo 'OK';
    }else{
        echo 'FAIL';
    }
    phpinfo();      //查看php信息
?>
```

### 8. 虚拟主机配置及wordpress安装

```bash
mkdir /web
tar xf wordpress-4.5.3-zh_CN.tar.gz -C /web
cd /web/wordpress/
cp wp-config{-sample,}.php
vim wp-config.php           # 编辑wp配置

    # 新建数据库，做如下修改并做相应授权，略
    define('DB_NAME', 'wordpress');     # 数据库名
    define('DB_USER', 'wpuser');        # 用户
    define('DB_PASSWORD', 'wppass');    # 密码

vim /etc/httpd/conf.d/vhost.conf        # 添加虚拟主机配置
    
    <VirtualHost 192.168.1.71:80>
        ServerName wp.liu2lin.com       # 服务器名
        DocumentRoot /web/wordpress     # 根
        <Directory "/web/wordpress">    # 授权
                Options None
                AllowOverride None
                Require all granted
        </Directory>
    </VirtualHost>
```
修改hosts文件添加: 192.168.1.71 wp.liu2lin.com，访问wp.liu2lin.com初始化wordpress站点

![wp](wordpress.png)

## 二、yum快速安装2，FastCGI方式(php-fpm)

说明：mariadb及httpd安装如上不再缀述
> httpd-2.2：默认不支持fcgi模块，需要自行编译扩展
> php-5.3.3：不支持fpm机制，需要自行打补丁编译安装

### 1. 安装

```
yum -y install httpd php-fpm php-mysql mariadb-server php-xcache
```

### 2. php-fpm

- **配置文件：**

> 服务进程配置文件：/etc/php-fpm.conf, /etc/php-fpm.d/\*.conf
> 解释器配置文件：/etc/php.ini, /etc/php.d/\*.ini

- **服务进程配置文件说明：**

```bash
[global]            # 全局配置

[pool]              # 连接池配置
listen = 127.0.0.1:9000     # 监听端口
listen.backlog = -1

listen.allowed_clients = 127.0.0.1      # 监听客户端请求，本实验中配置在同一主机

user = 
group = 

pm = dynamic|static     # 支持动态生成进程池和静态
pm.max_children = 50    # 最大线程数，即支持的最大并发
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests  = 500

# pm方式的php进程存储session的路径：
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session

# 如果目录不存在手动建立并修改属主
    # mkdir /var/lib/php/session
    # chown apache.apache /var/lib/php/session
```

- **配置虚拟主机**

```bash
vim /etc/httpd/conf.d/vhost.conf        # 添加虚拟主机配置
    
    <VirtualHost 192.168.1.71:80>
        ServerName wp.liu2lin.com       # 服务器名
        DocumentRoot /web/wordpress     # 根
        ProxyRequests Off               # 关闭正向代理
        DirectoryIndex index.php        # 索引文件
        ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/data/www/$1   # 设定代理
        <Directory "/web/wordpress">
                Options None
                AllowOverride None
                Require all granted
        </Directory>
    </VirtualHost>
```

- **启动服务：**

```bash
systemctl start php-fpm     # 启动php-fpm服务
ss -tuln sport = :9000      # 默认监听9000/tcp 
ps aux | grep php-fpm       # 查看进程
```

## 三、编译安装(在同一主机部署)

> **httpd：**httpd-2.4, 需要apr-1.4+及apr-util-1.4+以上依赖
> **mariadb：**mariadb-5.5, 二进制包安装
> **php：**php-5.5

### 开发包组及依赖包安装

```bash
    yum groupinstall 'Development Tools' 'Server Platform Development'
    yum install pcre-devel openssl-devel  libevent-devel                # httpd依赖
    yum install libxml2-devel gd-devel freetype-devel libmcrypt-devel   # php依赖
```

### httpd-2.4 编译
#### 1. apr及apr-util编译安装

```bash
tar xf apr-1.5.2.tar.bz2
cd apr-1.5.2
./configure --prefix=/usr/local/apr
make && make install

tar xf apr-util-1.5.4.tar.bz2
cd apr-util-1.5.4
./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr
make && make install

# 在centos7的yum源中apr及apr-utl版本符合要求，可不用编译安装
```
#### 2. httpd编译安装

```bash
tar xf httpd-2.4.23.tar.bz2
cd httpd-2.4.23
./configure --prefix=/usr/local/apache2 --sysconfdir=/etc/httpd2 --enable-so --enable-ssl --enable-cgi --enable-rewrite --with-zlib --with-pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util --enable-modules=most --enable-mpms-shared=all --with-mpm=event
make -j N       # N为物理核心的2倍，加速编译
make install
```

#### 3. 配置

```bash
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
# 如果php以模块方式结合 则为 
# LoadModule php5_module modules/libphp5-[zts].so

# 添加
AddType application/x-httpd-php  .php
AddType application/x-httpd-php-source  .phps

DirectoryIndex  index.php  index.html
```

#### 4. PATH，MAN, 头文件

```bash
echo 'export PATH=$PATH:/usr/local/apache2/bin' > /etc/profile.d/httpd.sh
vim /etc/man_db.conf  添加: MANDATORY_MANPATH  /usr/local/apache2/man     # centos7上只要导出PATH此步可省
ln -sv /usr/local/apache2/include /usr/include/apache   # 导出头文件
ldconfig 
```

#### 5. 启动

```bash
/usr/local/apache2/bin/apachectl start
ss -tlun | grep :80
ps aux | grep httpd
curl 192.168.1.72
```

### mariadb-5.5进制包安装

#### 1. 创建mysql用户
```bash
groupadd -r mysql
useradd -g mysql -r -s /sbin/nologin -M -d /mydata/ mysql
```

#### 2. 安装
```bash
tar xf mariadb-5.5.50-linux-x86_64.tar.gz -C /usr/local
cd /usr/local/
ln -sv mariadb-5.5.50-linux-x86_64  mysql   # 方便滚动升级
cd mysql 
chown -R mysql:mysql .
scripts/mysql_install_db --user=mysql --datadir=/mydata      # 初始化并指定数据存放路径
chown -R root .

# 注：数据建议存放在lvm上
```
#### 3. 配置

```bash
cp /usr/local/mysql/support-files/my-large.cnf  /etc/my.cnf
vim /etc/my.cnf

    [mysqld]
    thread_concurrency = 2      # cpu的物理核心*2
    datadir = /mydata           # 数据存放路径！！！
    innodb_file_per_table = On
    skip_name_resolve = On      # 禁止IP反解

cp /usr/local/mysql/support-files/mysql.server /etc/rc.d/init.d/mysqld  # 启动脚本
chmod +x /etc/rc.d/init.d/mysqld
```

#### 4. PATH路径,MAN文档,头文件

```bash
echo 'export PATH=$PATH:/usr/local/mysql/bin' > /etc/profile.d/mysql.sh      # PATH环境变量
vim /etc/man_db.conf  ==> MANDATORY_MANPATH  /usr/local/mysql/man               # 帮忙文档
ln -sv /usr/local/mysql/include  /usr/include/mysql                             # 头文件
echo '/usr/local/mysql/lib' > /etc/ld.so.conf.d/mysql.conf          # 输出mysql的库文件给系统库查找路径
ldconfig [-v]   # 系统重新载入系统库
```

#### 5. 启动服务

```
service mysqld start
ss -tlun | grep :80     
mysql       # 建议先执行mysql_secure_installation 进行安全设置
```

### php-5.5 编译安装

#### 1. 安装(php-fpm)

```bash
tar xf php-5.5.37.tar.bz2
cd php-5.5.37
./configure --prefix=/usr/local/php5 --with-mysql=/usr/local/mysql --with-openssl --with-mysqli=/usr/local/mysql/bin/mysql_config --enable-mbstring --enable-xml --enable-sockets --enable-fpm --with-freetype-dir --with-gd --with-libxml-dir=/usr --with-zlib --with-jpeg-dir --with-png-dir --with-mcrypt --with-config-file-path=/etc/php5.ini --with-config-file-scan-dir=/etc/php5.d --enable-maintainer-zts
make -j N       # N为物理核心的2倍，加速编译
make install

# 注：
# --enable-maintainer-zts                   # 为支持apache的worker或event
# --enable-fpm                              # 编译成fpm
# --with-apxs2=/usr/local/apache2/bin/apxs  # 编译成模块，跟fpm是冲突的
```

#### 2. 配置

```bash
cp ./php.ini-production /etc/php5.ini       # 源码目录下
cp /usr/local/php5/etc/php-fpm.conf.default /usr/local/php5/etc/php-fpm.conf 
vim /usr/local/php5/etc/php-fpm.conf

    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 2
    pm.max_spare_servers = 8
    pid = /usr/local/php5/var/run/php-fpm.pid    # 修改这绝对路径
```

#### 3. 服务脚本并启动

```bash
vim /etc/systemd/system/php-fpm.service
    
    [Unit]
    Description=The PHP FastCGI Process Manager
    After=syslog.target network.target

    [Service]
    Type=forking
    PIDFile=/usr/local/php5/var/run/php-fpm.pid
    ExecStart=/usr/local/php5/sbin/php-fpm --fpm-config /usr/local/php5/etc/php-fpm.conf
    ExecReload=/bin/kill -USR2 $MAINPID

    [Install]
    WantedBy=multi-user.target

systemctl start php-fpm
ss -tunl | grep :9000
```

#### 4. httpd设置代理

```bash
vim /etc/httpd/httpd.conf
    
    DirectoryIndex index.php index.html
    ProxyRequests Off           # 关闭正向代理
    ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/usr/local/apache2/htdocs/$1

# 注：建议在虚拟机中设置

/usr/local/apache2/bin/apachectl restart
systemctl restart php-fpm
```

#### 5. 安装xcache加速
        
```bash
tar
cd
/usr/local/php5/bin/phpize      # 添加php扩展时在源码目录下执行
./configure --enable-xcache --with-php-config=/usr/local/php5/bin/php-config
make && make install

# 注：执行安装后出现  Installing shared extensions:     /usr/local/php5/lib/php/extensions/no-debug-zts-20121212/

mkdir /etc/php5.d/
cp xcache.ini  /etc/php5.d/     # 复制配置文件并查看如下值
    
    extension = xcache.so   # 确保开启
    xcache.size  =  160M    # 适当调整缓存大小

systemctl restart php-fpm
```

### 测试

```bash
echo '<?php phpinfo();?>' > /usr/local/apache2/htdocs/index.php
curl 192.168.1.72
```

*虚拟主机设置同上，此略*


## 四. 基于FastCGI不同主机分别编译安装

### 1. 环境准备
- httpd-2.4：192.168.1.73
- php-5.5：192.168.1.74
- mariadb-5.5：192.168.1.75

**httpd和mariadb编译方式同上，则处编译php**

### 2. php安装
```bash
yum install libxml2-devel gd-devel freetype-devel libmcrypt-devel openssl-devel bzip2-devel
tar
cd
./configure --prefix=/usr/local/php5 --with-mysql=mysqlnd --with-pdo-mysql=mysqlnd --with-mysqli=mysqlnd --with-openssl --enable-mbstring --with-freetype-dir --with-jpeg-dir --with-png-dir  --with-libxml-dir=/usr/ --enable-xml --enable-sockets --enable-fpm --with-config-file-path=/etc/ --with-config-file-scan-dir=/etc/php5.d --with-bz2 --enable-maintainer-zts
make -j N    # N为物理核心的2倍，加速编译
make install
```
### 3. 配置
- php配置

```bash
cp ./php.ini-production /etc/php5.ini       # 源码目录下
cp /usr/local/php5/etc/php-fpm.conf.default /usr/local/php5/etc/php-fpm.conf 
vim /usr/local/php5/etc/php-fpm.conf

    listen 192.168.1.74:9000       # 修改为php-fpm服务监听地址！！！！！
    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 2
    pm.max_spare_servers = 8
    pid = /usr/local/php5/var/run/php-fpm.pid    # 修改这绝对路径
```

- httpd配置：/etc/httpd2/httpd.conf

```bash
# 开启如下模块
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
# 如果php以模块方式结合 则为 
# LoadModule php5_module modules/libphp5-[zts].so

# 添加
AddType application/x-httpd-php  .php
AddType application/x-httpd-php-source  .phps
```
- httpd虚拟机配置：/etc/httpd2.d/vhost.conf (手动建立)

```bash
<VirtualHost 192.168.1.73:80>
    ServerName www3.liu2lin.com
    DocumentRoot /web
    ProxyRequests Off         
    DirectoryIndex index.php 
    ProxyPassMatch ^/(.*\.php)$ fcgi://192.168.1.74:9000/web/$1
    <Directory "/web">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

- 测试mysql连接性  

> 在php-fpm(192.168.1.74)上新建测试页/web/index.php，mysql先授权，参考上，然后浏览器访问192.168.1.73查看结果

```php
<?php
    $con = mysql_connect('192.168.1.75','wpuser','wppass');
    if($con){
        echo 'OK';
    }else{
        echo 'FAIL';    
```

- 虚拟主机搭建phpMyAdmin

```bash
# php-fpm服务器上
unzip phpMyAdmin-4.6.3-all-languages.zip
mv phpMyAdmin-4.6.3-all-languages /web/phpmyadmin
cd /web/phpmyadmin
cp config.sample.inc.php config.inc.php     # 配置文件
vim config.inc.php  # 修改以下

    $cfg['blowfish_secret'] = 't4hjJbhU1iLz6g';     # cookie加密
    $cfg['Servers'][$i]['host'] = '192.168.1.75';   # mysql服务器地址

# 加密码最好使用openssl rand -base64 10  来生成
```

```bash
# httpd服务器上虚拟主机设置
<VirtualHost 192.168.1.73:80>
    ServerName pma3.liu2lin.com
    DocumentRoot /web/phpmyadmin
    ProxyRequests Off
    DirectoryIndex index.php
    ProxyPassMatch ^/(.*\.php)$ fcgi://192.168.1.74:9000/web/phpmyadmin/$1
    <Directory "/web/phpmyadmin">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>

# httpd主机上需要建立/web/phpmyadmin，否则将出现403错误 !!!!
```

> 注：php-fpm服务器只负责解析php文件，html,图片等静态资源仍需由httpd来解析，如下

![pma1](pma1.png)

> 为方便起见直接将phpMyAdmin目录复制到/web/下并，图片可正常解析

![pma2](pma2.png)

## 压测结果

....










