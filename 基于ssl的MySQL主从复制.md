---
title: 基于ssl的MySQL主从复制
date: 2016-09-02 20:20:40
categories: Linux服务
tags: [MySQL主从, ssl]
---

本文简要自建CA中心用于基于ssl的MySQL的主从复制的实现
<!--more-->

## 环境准备

- 系统：CentOS_7.2_x86_64
- 程序：mariadb-5.5.50-1.el7_2.x86_64
- CA中心服务器：172.16.60.71
- 主服务器：172.16.60.3
- 从服务器：172.16.60.4

## CA中心服务器搭建
![CA](ca.png)

## 主从服务器申请证书
1. 主从服务器申请证书
    ![master_ca](master_ca.png)

2. 从服务器申请参考主服务器，将生成申请文件改为slave.csr即可

## CA签发发证书
1. 为主服务器签发证书
    ![ca_2](ca_2.png)

2. 为从服务器签发同上

## 主从服务配置
1. 主服务器配置
    `vim /etc/my.cnf`
        [mysqld]
        log_bin = log-bin
        server_id = 1
        innodb_file_per_table=ON
        skip_name_resolve=ON

        ssl
        ssl-ca = /etc/my.cnf.d/ssl/cacert.pem
        ssl-cert = /etc/my.cnf.d/ssl/master.crt
        ssl-key = /etc/my.cnf.d/ssl/master.key

2. 从服务器配置
        [mysqld]
        log_relay = log-relay
        server_id = 2
        read_only=ON
        innodb_file_per_table=ON
        skip_name_resolve=ON

        ssl
        ssl-ca = /etc/my.cnf.d/ssl/cacert.pem
        ssl-cert = /etc/my.cnf.d/ssl/slave.crt
        ssl-key = /etc/my.cnf.d/ssl/slave.key

3. 修改密钥及证书权限(主从服务器都须修改)
    ![chown](chown.png)

## 主服务器授权
启动mariadb服务并登录进行如下设置

    # 查看ssl是否启用及相关信息
    MariaDB [(none)]> SHOW GLOBAL VARIABLES LIKE '%ssl%';
    +---------------+------------------------------+
    | Variable_name | Value                        |
    +---------------+------------------------------+
    | have_openssl  | YES                          |
    | have_ssl      | YES                          |
    | ssl_ca        | /etc/my.cnf.d/ssl/cacert.pem |
    | ssl_capath    |                              |
    | ssl_cert      | /etc/my.cnf.d/ssl/master.crt |
    | ssl_cipher    |                              |
    | ssl_key       | /etc/my.cnf.d/ssl/master.key |
    +---------------+------------------------------+

    # 授权信息
    MariaDB [(none)]> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO 'repuser'@'172.16.60.%' IDENTIFIED BY 'reppass' REQUIRE ssl;
    MariaDB [(none)]> FLUSH PRIVILEGES;
    
    # 查看二进制日志
    MariaDB [(none)]> SHOW MASTER STATUS;
    +----------------+----------+--------------+------------------+
    | File           | Position | Binlog_Do_DB | Binlog_Ignore_DB |
    +----------------+----------+--------------+------------------+
    | log-bin.000001 |      507 |              |                  |
    +----------------+----------+--------------+------------------+

## 从服务器命令行登录测试
    mysql -h172.16.60.3 -urepuser -p --ssl-ca=/etc/my.cnf.d/ssl/cacert.pem --ssl-cert=/etc/my.cnf.d/ssl/slave.crt --ssl-key=/etc/my.cnf.d/ssl/slave.key

## 从服务器设置
启动服务并配置为从服务器

    # 查看ssl是否启用及相关信息
    MariaDB [(none)]> SHOW GLOBAL VARIABLES LIKE '%ssl%';
    +---------------+------------------------------+
    | Variable_name | Value                        |
    +---------------+------------------------------+
    | have_openssl  | YES                          |
    | have_ssl      | YES                          |
    | ssl_ca        | /etc/my.cnf.d/ssl/cacert.pem |
    | ssl_capath    |                              |
    | ssl_cert      | /etc/my.cnf.d/ssl/slave.crt  |
    | ssl_cipher    |                              |
    | ssl_key       | /etc/my.cnf.d/ssl/slave.key  |
    +---------------+------------------------------+

    # 设置主从
    MariaDB [(none)]> CHANGE MASTER TO 
      -> MASTER_HOST='172.16.60.3',
      -> MASTER_USER='repuser',
      -> MASTER_PASSWORD='reppass',
      -> MASTER_LOG_FILE='log-bin.000001',
      -> MASTER_LOG_POS=507,
      -> MASTER_SSL=1,
      -> MASTER_SSL_CA='/etc/my.cnf.d/ssl/cacert.pem',
      -> MASTER_SSL_CERT='/etc/my.cnf.d/ssl/slave.crt',
      -> MASTER_SSL_KEY='/etc/my.cnf.d/ssl/slave.key';

    # 开启从服务器IO及SQL线程
    MariaDB [(none)]> START SLAVE;
    MariaDB [(none)]> SHOW SLAVE STATUS\G;
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 172.16.60.3
                      Master_User: repuser
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: log-bin.000001
              Read_Master_Log_Pos: 507
                   Relay_Log_File: log-relay.000001
                    Relay_Log_Pos: 621
            Relay_Master_Log_File: log-bin.000001
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: 
              Replicate_Ignore_DB: 
               Replicate_Do_Table: 
           Replicate_Ignore_Table: 
          Replicate_Wild_Do_Table: 
      Replicate_Wild_Ignore_Table: 
                       Last_Errno: 0
                       Last_Error: 
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 339
                  Relay_Log_Space: 909
                  Until_Condition: None
                   Until_Log_File: 
                    Until_Log_Pos: 0
               Master_SSL_Allowed: Yes
               Master_SSL_CA_File: /etc/my.cnf.d/ssl/cacert.pem
               Master_SSL_CA_Path: 
                  Master_SSL_Cert: /etc/my.cnf.d/ssl/slave.crt
                Master_SSL_Cipher: 
                   Master_SSL_Key: /etc/my.cnf.d/ssl/slave.key
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error: 
                   Last_SQL_Errno: 0
                   Last_SQL_Error: 
      Replicate_Ignore_Server_Ids: 
                 Master_Server_Id: 1

## 测试
在主服务器上进行数据CURD操作，查看从服务器同步情况


以上就是简单配置的整个过程，本实验中数据库为全新安装，所有没涉及主服务器导出数据，从服务器导入过程

