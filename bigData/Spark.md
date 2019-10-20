# Spark

## 编译源码

### 使用maven编译

1.  安装好java8, maven3.5.4以上版本

2.  可选: 设置maven所使用的内存

    1.  ```shell
        export MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=512m"
        ```

3.  编译为一个即拆即用的压缩包

    ```shell
    ./dev/make-distribution.sh --name custom-spark --tgz -Psparkr -Phadoop-2.7 -Phive -Phive-thriftserver -Pmesos -Pyarn -Pkubernetes -Dhadoop.version=2.6.0-cdh5.7.0
    # -Phadoop-2.x/3.x 指定hadoop版本 
    #只用cdh hadoop需要在pom.xml添加
    <repository>
       <id>cloudera</id>       
           <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
    </repository>
    ```

4.  直接编译

    ```shell
    ./build/mvn -Pyarn -Phadoop-2.7 -Dhadoop.version=2.7.3 -DskipTests clean package
    # 需要添加哪些模块则在前面添加 -Pxxx例如 -Phive
    ```

5.  修改scala版本

    ```shell
    ./build/mvn -Pscala-2.12 compile
    ./dev/change-scala-version.sh 2.12
    ./build/sbt -Dscala.version=2.12.6
    ```

    

## 配置Spark

1.  启动spark Local模式:

    ```shell
    ./bin/pyspark --master local[2] #bash
    ./bin/pyspark --master "local[2]" #zsh
    ```

    

2.  启动Standalone模式

    1.  master + n worker

    2.  在conf/spark-env.sh中编辑

        ```shell
        SPARK_MASTER_HOST=localhost
        SPARK_WORKER_CORES=2
        SPARK_WORKER_MEMORY=2g
        ```

    3.  在spark中启动/sbin中启动./start-all.sh

    4.  启动spark-shell 

        ```shell
        ./bin/pyspark --master "spark://localhost:7077" #zsh
        ```

        

3.  wordCount例子

    ```scala
    val file = spark.sparkContext.textFile("file:///home/eugeo/app/data/hello.txt")
    val wordCounts = file.flatMap(line => line.split(" ")).map((word => (word, 1))).reduceByKey(_ + _)
    wordCounts.collect
    ```

    

## Spark SQL

1.  ###概述

    1.  用来处理结构化数据
    2.  Spark SQl 应用不局限于SQL
    3.  可以访问hive, json, parquet等文件数据
    4.  SQL只是Spark SQL的一个功能
    5.  xx提供了SQL的api, DataFrame和Dataset的API

2.  ## 使用

    1.  **与Hive联用需要将Hive-site.xml 复制到 ~/conf下**
    
        1.  使用:
    
            1.  ```scala
                spark.sql("select * from xxx").show
                ```
    
        2.  使用spark-sql
    
            1.  ```SPARQL
                select * from xxx
                ```
    
    2.  **使用thriftservcer**
    
        1.  ```shell
            ./sbin/start-thriftserver.sh #与spark-shell spark-sql用法相同
            ./sbin/start-thriftserver.sh --help #获取帮助
            ```
    
        2.  默认端口为10000
    
        3.  使用beeline
    
            ```shell
            ./bin/beeline
            beeline> !connect jdbc:hive2://localhost:10000
            ```
    
        4.  与spark-shell/spark-sql区别:
    
            1.  spark-shell, spark-sql都是一个spark application
            2.  thriftserver, 不管启动多少客户端(beeline\ code), 永远是一个spark application
            3.  解决了数据共享问题, 多个客户端可以共享数据
        
    3.  **使用jdbc访问**

##  DataFrame&DataSet

###  DataFrame&DataSet概述

1.  DataSet是分布式的数据集
2.  DataFrame是以列的形式构成的分布式数据集

