---
layout: post
title:  "搭建 Hadoop 以及 HBase 开发环境"
date:   2015-04-10 18:00:00
tags:
    - Hadoop
    - HBase
---

# Hadoop 

##### [来源1][Hadoop1] [来源2][Hadoop2]

### 配置:
  etc/hadoop/core-site.xml:

    <configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://localhost:9000</value>
        </property>
    </configuration>

  etc/hadoop/hdfs-site.xml:
    
    <configuration>
      <property>
          <name>dfs.replication</name>
          <value>1</value>
      </property>
    </configuration>


### 一些命令
  格式化文件系统:

    $ bin/hdfs namenode -format

  启动 NameNode 进程和 DataNode 进程:

    $ sbin/start-dfs.sh

  进程日志写在 `$HADOOP_LOG_DIR` 目录下 (默认是 `$HADOOP_HOME/logs`).

  访问 NameNode 的网络界面，默认是:

    http://localhost:50070/

  创建文件夹等，为 HDFS 执行 MapReduce 任务的准备:

    $ bin/hdfs dfs -mkdir /user
    $ bin/hdfs dfs -mkdir /user/<username>

  复制文件到分布式文件系统:

    $ bin/hadoop fs -put <localsrc> ... <dst>

  从分布式文件系统拷贝文件到本地:

    $ bin/hdfs dfs -get output output
    $ cat output/*

  或直接查看分布式文件系统的输出文件:

    $ bin/hdfs dfs -cat output/*

  关闭 Hadoop 进程:

    $ sbin/stop-dfs.sh

# [Hbase][Hbase]

  启动: 

    $ /bin/start-hbase.sh

  停止: 

    $ /bin/stop-hbase.sh

  Shell: 

    $ /bin/hbase shell

  Shell 命令: 
    
    list, scan 'TableName', disable 'TableName', drop 'TableName'

[Hadoop1]:https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html
[Hadoop2]:http://hadoop.apache.org/docs/r1.0.4/cn/hdfs_shell.html
[Hbase]:http://hbase.apache.org/book.html#quickstart
