---
layout: post
title: 浅谈数据库读写分离  
categories: MySQL
description: write read divide
keywords: write read divide
---



### 1. 优点

##### 1.1 让读库分担 主库的读压力，
##### 1.2 提高性能
    
    1.2.1 主从只负责各自的写和读，极大程度的缓解X锁和S锁争用
    1.2.2 从库可配置myisam引擎，提升查询性能以及节约系统开销
##### 1.3 增加冗余

### 2. 需要考虑的问题：

##### 2.1 数据库复制问题

MySQL同步的流程大致如下: 
1. 主服务器(master)将变更事件(更新、删除、表结构改变等等)写入二进制日志(master log)。 
2. 从服务器(slave)的IO线程从主服务器(binlog dump线程)获取二进制日志，并在本地保存一份自己的二进制日志(relay log) 
3. 从服务器的SQL线程读取本地日志(relay log)，并重演变更事件。

##### 2.2对数据源的选择问题

应用程序与ＤＢ之间有个中央控制台服务器，根据负载均衡策略决定访问哪一台DB服务器

### 3. 如何实现读库读，主库写

MySQL Proxy基于MySQL Client 和 MySQL Server之间的代理程序，能够完成对Client所发请求的监控、修改。从Client角度看，通过Proxy访问Server和直接访问Server没有任何区别。对于既有的程序而言，只要把直接被访问的Server的IP地址和端口号换成Proxy的IP地址和端口号就可以。

MySQL Proxy的工作原理也较简单。在Proxy启动时可以指定Proxy所需要使用的lua脚本，在lua脚本中预先实现6个方法： 

    * connect_server()     // 接收到Client的连接请求时调用 
    * read_handshake()   // 
    * read_auth()               // 读取Client的认证信息时调用 
    * read_auth_result() // 读取认证结果时调用 
    * read_query()            // 读取Client的query请求时调用 
    * read_query_result()   //读取query结果时调用 

当 Proxy接收到Client请求时，在请求的不同的阶段会调用上面的不同方法。这样Proxy使用者就可以根据自己的业务需求，自由的实现这6个方法达到目的。 

通过在read_query()中加入代码，我们可以截取出当前的请求是insert、update还是select，然后把 insert和update请求发送到Master中，把select请求发送到Slave中，这样就解决了读写分离的问题。

### 4. 解决主从延迟方案

##### 4.1 MySQL Proxy
在Master上增加一个自增表，这个表仅含有1个的字段。当Master接收到任何数据更新的请求时，均会触发这个触发器，该触发器更新自增表中的记录。
由于Count_table也参与Mysq的主从同步，因此在Master上作的 Update更新也会同步到Slave上。当Client通过Proxy进行数据读取时，Proxy可以先向Master和Slave的 Count_table表发送查询请求，当二者的数据相同时，Proxy可以认定 Master和Slave的数据状态是一致的，然后把select请求发送到Slave服务器上，否则就发送到Master上。

##### 4.2 用redis缓存

### 5. 疑问：
##### 5.1 如何实现主服务器 把 binlog传输到从服务器？
5.1.1 MySQL的同步机制是依赖Slave主动向Master发请求来获取数据的。
从服务器配置主服务器的ip，账号密码等，通过mysql的语句进行复制,如

```
# 执行同步命令，设置主数据库ip，同步帐号密码，同步位置 
mysql>change master to master_host='192.168.1.2',master_user='slave',master_password='111111',master_log_file='mysql-bin.000009',master_log_pos=196; 
 
# 开启同步功能 
mysql>start slave;
```

5.1.2 百度用bigpine消息队列 进行复制

##### 5.2 主服务器 复制binlog到从服务器时，是否需要对binlog进行加锁？这个性能消耗不是很大?
目前的了解是需要加锁，可以选择n个事务后进行一次 binlog存储