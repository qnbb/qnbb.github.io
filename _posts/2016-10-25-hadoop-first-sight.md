---
layout: post
title: 初识 Hadoop
categories: Hadoop
description: Hadoop
keywords: Hadoop
---
 
### 一. Hadoop 前世今生
Google大数据技术
MapReduce、BigTable 、GFS
1. 成本降低，能用PC机，就不用大型机和高端存储
2. 软件容错硬件故障视为常态，通过软件保证可靠性
3. 简化并行分布式计算，无须控制节点同步、消息通讯、数据交换，只需要Map和Reduce

一个模仿Google的大数据技术的开源实现 Hadoop。

---------------

### 二. Hadoop 的功能与优势

#### Hadoop 是什么
Apache 开源 分布式存储 ＋ 分布式计算平台
包括两个核心组成：
1. HDFS：分布式文件系统，存储海量的数据
2. MapReduce：并行处理框架，实现任务分解和调度

#### Hadoop 可以用来做什么？
搭建大型数据仓库，PB级数据的存储、处理、分析、统计等业务
如：搜索引擎、网页处理、日志分析、商业智能、数据挖掘
#### Hadoop 优势
1. 高扩展
2. 低成本
3. 成熟的生态圈（Hbase, HIVE ...）

#### Hadoop 应用情况
除了 Google、微软 的大部分公司

---------------

### 三. Hadoop 生态系统及版本
#### 生态系统
除了 HDFS、MapReduce，还有
1. HIVE，将SQL语句转换为 Hadoop任务，降低使用Hadoop的门槛
2. HBase，放弃事物特性，追求更高的扩展（VS - 传统数据库）；提供数据的随机读写和实时访问，实现对表数据的读写功能（VS - HDFS）
3. zookeeper，监控Hadoop 集群里每个节点的状态、管理整个集群的配置、维护节点之间的数据一致性

####版本
ver1.x（ver1.2 稳定版本）
ver2.x

---------------

### 四. Hadoop安装
Step 1:准备Linux环境
Step 2:安装JDK
Step 3:配置Hadoop
wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz

[配置](http://www.imooc.com/video/7722)：
/etc/profile 配置 HADOOP_HOME, JAVA_HOME
conf 文件下，
mapred-site.xml
core-site.xml
hdfs-site.xml
hadoop-env.sh
