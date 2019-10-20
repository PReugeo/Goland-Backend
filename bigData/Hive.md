# 大数据数据仓库Hive

## 概述

1.  由FaceBook开源, 最初用于解决海量结构化日志数据统计问题
2.  构建在Hadoop之上的数据仓库
3.  Hive定义了一种类SQL查询语言:HQL
4.  通常用于进行离线数据处理(MapReduce)
5.  底层支持多种不同的执行引擎(MapReduce, Tez, Spark)
6.  支持多种不同压缩格式, 存储格式和自定义函数
    1.  压缩:GZIP, LZP, Snappy, BZIP
    2.  存储:TextFile, SequenceFile, RCFile, ORC, Parquet
    3.  UDF: 自定义函数

## 环境搭建

### 下载地址

>   ​	http://archive.cloudera.com/cdh5/cdh/5/

### 配置Hive

hive-site.xml  

```xml
<configuration>
    <!-- 配置数据库连接参数 -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mariadb://localhost:3306/sparkSQL?createDatabaseIfNotExist=true</value>
    </property>
    <!-- 连接数据库驱动 -->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.mariadb.jdbc.Driver</value>
    </property>
    <!-- 连接数据库用户名 -->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <!-- 密码 -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>1212</value>
    </property>
</configuration>
```

*注:* 需要将连接数据库的驱动包放在Hive安装目录下的/lib下

### 创建表

```HQL
CREATE TABLE table_name
​	[(col_name data_type [COMMENT col_comment])]
```



#### 加载数据到hive表中

``` HQL
LOAD DATA LOCAL INPATH 'filepath' INTO TABLE tablename
```

