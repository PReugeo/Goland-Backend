# Hadoop 基础知识

### MapReduce和传统关系型数据库比较

1.  数据大小   PB											GB
2.  数据存取   批处理                                     交互式和批处理
3.  更新            一次写入, 多次读取               多次读/写
4.  事物           无                                             ACID
5.  结构           读时模式                                 写时模式
6.  完整性        低                                            高
7.  横向扩展    线性的                                     非线性的



### Hadoop安装

**CDH下载: https://archive.cloudera.com/cdh5/cdh/5/**

```bash
tar -xzf hadoop-2.6.0-cdh5.7.0.tar.gz #解压缩
export HADOOP_HOME=xxxxx  #注册hadoop的环境变量
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin #注册hadoop可执行文件的目录
hadoop version #检验hadoop环境变量是否设置成功
```



### Hadoop配置

Hadoop各个组件均可以在XML文件中配置。*core-site.xml*配置通用属性。 *hdfs-site.xml*文件配置HDFS属性。*mapred-site.xml*配置MapReduce. *yarn-site.xml*配置yarn属性. 文件均在*/etc/hadoop*子目录中



默认配置位于Hadoop安装路径 share/doc 下四个HTML文件中



### Hadoop运行模式

1.  独立(本地)模式: 无需运行任何守护进程, 所有程序均在一个JVM中运行, 适合开发阶段
2.  伪分布模式: Hadoop守护进程运行在本地机器上, 模拟小集群
3.  全分布模式: Hadoop守护进程运行在一个集群上.



在分布模式下启动HDFS和YARN守护进程, 还需要配置MapReduce以便使用YARN

<!--不同模式的关键配置属性-->

| 组件名称  | 属性名称                      | 独立模式 | 伪分布模式        | 全分布模式        |
| --------- | ----------------------------- | -------- | ----------------- | ----------------- |
| Common    | fs.defaultFS                  | file://  | hdfs://localhost/ | hdfs://namenode/  |
| HDFS      | dfs.replication               | N/A      | 1                 | 3(默认)           |
| MapReduce | mapreduce.framework.name      | local    | yarn              | yarn              |
| YARN      | yarn.resourcemanager.hostname | N/A      | Localhost         | resourcemanager   |
|           | yarn.nodemanager.aux-services | N/A      | mapreduce_shuffle | mapreduce_shuffle |



### 伪分布模式设置

1.  创建各个配置文件到Hadoop安装路径/etc/hadoop. 或者把/etc/hadoop目录复制到另一个位置, 然后把**-site.xml*这些配置文件放在该目录下.

    第二种方法优点: 将配置设置和安装文件隔离开.

    要将环境变量**HADOOP_CONF_DIR **设置成指向那个新目录, 或确定在启动守护进程时使用 - -config选项

```xml
<?xml version="1.0"?>
<!-- core-site.xml -->
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost/</value>
    </property>
 
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/eugeo/app/tmp</value>
    </property>
</configuration>
```

2.  配置ssh

    hadoop不严格区分全分布模式和伪分布模式, 所以用伪分布模式时要用ssh连接本地(localhost)

    ```bash
    #基于空口令生成一个ssh秘钥
    ssh-keygen -t rsa -p '' -f ~/.ssh/id_rsa
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    #测试
    ssh localhost
    ```

3.  格式化HDFS文件系统

    ```bash
    #首次使用Hadoop, 必须格式化文件系统 ~/bin
    hdfs namenode -format
    #启动HDFS, YARN, MapReduce ~/sbin
    start-dfs.sh
    start-yarn.sh
    mr-jobhistory-daemon.sh start historyserver
    #若配置文件不在默认conf目录中, 则需要运行脚本前使用HADOOP_CONF_DIR 或启动守护进程时使用--config选项
    mr-jobhistory-daemon.sh --config path-to-config-directory start historyserver
    
    #验证是否成功
    jps
    	DataNode
    	SecondaryNameNode
    	NameNode
    #浏览器方式 localhost:50070 8088
    ```

4.  创建用户目录

    hadoop fs -mkdir -p /user/$USER

5.  关闭守护进程

    ```bash
    mr-jobhistory-daemon.sh stop historyserver
    stop-dfs.sh
    stop-yarn.sh
    ```




### 分布式环境搭建

1.  先获取到分布式集群服务器的IP

    1.  ```shell
        hadoop000: 192.168.199.102
        hadoop001: 192.168.199.247
        hadoop002: 192.168.199.135
        ```

2.  hostname设置： sudo vim /etc/sysconfig/network

    1.  ```shell
        NETWORKING=yes
        HOSTNAME=xxxxx
        ```

3.  hostname和ip地址的设置 sudo vim /etc/hosts

    1.  ```shell
        192.168.199.102 hadoop000
        192.168.199.247 hadoop001
        192.168.199.135 hadoop002
        ```

4.  设置一个主节点(各节点角色分配)

    1.   hadoop000: NameNode ResourceManager
    2.   hadoop001: DataNode   NodeManager
    3.   hadoop002: DataNode   NodeManager

#### 前置配置

1.  **配置ssh（免密码登录)**

    1.  ```shell
        #基于空口令生成一个ssh秘钥
        ssh-keygen -t rsa -p '' -f ~/.ssh/id_rsa #每一台运行
        #以hadoop000机器为主
        ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop000
        ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop001
        ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop002
        #测试
        ssh hadoop000
        ssh hadoop001
        ssh hadoop002
        ```


2.  **配置JDK**

    ```shell
    下载JAVA JDK
    export JAVA_HOME=~/app/xxx
    export PATH=$JAVA_HOME/BIN:$PATH
    ```

3.  **安装hadoop(CDH)**

    ```shell
    export HADOOP_HOME=~/app/xxx
    export PATH=$HADOOP_HOME:$PATH#
    ```

4.  **配置slaves和相关配置文件**

    ```shell
    #slaves
    hadoop000
    hadoop001
    hadoop002
    ```

    

### 分发安装包

```shell
scp -r ~/app hadoop@hadoop001:~/
scp -r ~/app hadoop@hadoop002:~/

scp ~/.bash_profile hadoop@hadoop001:~/
scp ~/.bash_profile hadoop@hadoop002:~/

#source ~/.bash_profile in hadoop001 and hadoop002

#首次使用Hadoop, 必须格式化文件系统 ~/bin在hadoop000上执行
hdfs namenode -format
```



### CDH在IDEA中使用

1.  使用maven进行包管理

2.  ```xml
  <!--cdh包需要添加仓库-->
    <repositories>
	<repository>
            <id>cloudera</id>            		       <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
    </repository>
  </repositories>
  
  <!--cdh包需要添加依赖-->
  
      <properties>
          <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
          <hadoop.version>2.6.0-cdh5.7.0</hadoop.version>
      </properties>
  
      <dependencies>
  
          <dependency>
              <groupId>org.apache.hadoop</groupId>
              <artifactId>hadoop-client</artifactId>
              <version>${hadoop.version}</version>
          </dependency>
  
          <dependency>
              <groupId>junit</groupId>
              <artifactId>junit</artifactId>
              <version>4.10</version>
              <scope>test</scope>
          </dependency>
  
          <!--添加spring for hadoop 依赖-->
          <dependency>
              <groupId>org.springframework.data</groupId>
              <artifactId>spring-data-hadoop</artifactId>
              <version>2.5.0.RELEASE</version>
          </dependency>
      </dependencies>
  ```
  
  
  

Q: 为什么在core-site.xml中设置副本系数为1, 查询到的确实3呢?

A: 如果使用hadoopshell上传的(put), 才采用默认的副本系数1

​	如果使用java API上传的, 在本地我们没有手动设置副本系数, 所以采用的是hadoop自己的默认系数3

### 使用

```shell
hadoop fs -copyFromLocal xxx \ xxx #从本地文件系统上传至HDFS
hadoop fs -copyToLocal
hadoop fs -ls
```

### 使用log4j

在classpath下新建**log4j.properties**

```properties
log4j.rootLogger=INFO,logfile,stdout
            
#log4j.logger.org.springframework.web.servlet=INFO,db

#log4j.logger.org.springframework.beans.factory.xml=INFO
#log4j.logger.com.neam.stum.user=INFO,db

#log4j.appender.stdout=org.apache.log4j.ConsoleAppender
#log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
#log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH\:mm\:ss} %p [%c] %X{remoteAddr}  %X{remotePort}  %X{remoteHost}  %X{remoteUser} operator\:[\u59D3\u540D\:%X{userName} \u5DE5\u53F7\:%X{userId}] message\:<%m>%n

#write log into file
log4j.appender.logfile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.logfile.Threshold=warn
log4j.appender.logfile.File=${webapp.root}\\logs\\main.log
log4j.appender.logfile.DatePattern=.yyyy-MM-dd
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=[AppLog] %d{yyyy-MM-dd HH\:mm\:ss} %X{remoteAddr} %X{remotePort} %m %n


#display in console
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Threshold=info
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[AppLog] %d{yyyy-MM-dd HH\:mm\:ss} %X{remoteAddr} %X{remotePort} %m %n
```

