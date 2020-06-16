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

1.  ### 概述

    1.  用来处理结构化数据
    2.  Spark SQL 应用不局限于SQL
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
            
            4.  在使用jdbc开发时，一定要先启动thriftserver
            
                ```scala
                Exception in thread "main" java.sql.SQLException: 
                Could not open client transport with JDBC Uri: jdbc:hive2://hadoop001:14000: 
                java.net.ConnectException: Connection refused
                ```
            
                
        
    3.  **使用jdbc访问**
    
    4.  **提交spark Application到环境中运行**
    
        ```shell
        spark-submit \
        --name SQLContextApp \
        --class com.imooc.spark.SQLContextApp \
        --master local[2] \
        /home/hadoop/lib/sql-1.0.jar \
        /home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/src/main/resources/people.json
        ```
    
        

##  DataFrame&DataSet

###  DataFrame&DataSet概述

1.  DataSet是分布式的数据集
2.  DataFrame是以列的形式构成的分布式数据集

### DataFrame Api操作

```scala
val spark = SparkSession.builder().appName("DataFrameApp").master("local[2]").getOrCreate()

// 将json文件加载成一个dataframe
val peopleDF = spark.read.format("json").load("file:///Users/rocky/data/people.json")

// 输出dataframe对应的schema信息
peopleDF.printSchema()

// 输出数据集的前20条记录
peopleDF.show()

//查询某列所有的数据： select name from table
peopleDF.select("name").show()

// 查询某几列所有的数据，并对列进行计算： select name, age+10 as age2 from table
peopleDF.select(peopleDF.col("name"), (peopleDF.col("age") + 10).as("age2")).show()

//根据某一列的值进行过滤： select * from table where age>19
peopleDF.filter(peopleDF.col("age") > 19).show()

//根据某一列进行分组，然后再进行聚合操作： select age,count(1) from table group by age
peopleDF.groupBy("age").count().show()

spark.stop()
```

```scala
/**
 * DataFrame中的操作操作
 */
object DataFrameCase {

  def main(args: Array[String]) {
    val spark = SparkSession.builder().appName("DataFrameRDDApp").master("local[2]").getOrCreate()

    // RDD ==> DataFrame
    val rdd = spark.sparkContext.textFile("file:///Users/rocky/data/student.data")

    //注意：需要导入隐式转换
    import spark.implicits._
    val studentDF = rdd.map(_.split("\\|")).map(line => Student(line(0).toInt, line(1), line(2), line(3))).toDF()

    //show默认只显示前20条
    studentDF.show
    studentDF.show(30)
    //show显示30条
    studentDF.show(30, false)

    //return Array()
    studentDF.take(10)
    studentDF.first()
    studentDF.head(3)


    studentDF.select("email").show(30,false)


    studentDF.filter("name=''").show
    studentDF.filter("name='' OR name='NULL'").show


    //name以M开头的人
    studentDF.filter("SUBSTR(name,0,1)='M'").show
     = studentDF.filter("substring(name,0,1)='M'").show

    studentDF.sort(studentDF("name")).show
    studentDF.sort(studentDF("name").desc).show

    studentDF.sort("name","id").show
    studentDF.sort(studentDF("name").asc, studentDF("id").desc).show

    studentDF.select(studentDF("name").as("student_name")).show


    val studentDF2 = rdd.map(_.split("\\|")).map(line => Student(line(0).toInt, line(1), line(2), line(3))).toDF()

    studentDF.join(studentDF2, studentDF.col("id") === studentDF2.col("id")).show

    spark.stop()
  }

  case class Student(id: Int, name: String, phone: String, email: String)

}
```





### DataFrame和RDD互操作

1.  第一种方式: 反射: case class 前提:事先知道你的字段,字段类型
2.  编程:Row 

```scala
/**
 * DataFrame和RDD的互操作
 */
object DataFrameRDDApp {

  def main(args: Array[String]) {

    val spark = SparkSession.builder().appName("DataFrameRDDApp").master("local[2]").getOrCreate()

    //inferReflection(spark)

    program(spark)

    spark.stop()
  }

  def inferReflection(spark: SparkSession) {
    // RDD ==> DataFrame第一种方式
    val rdd = spark.sparkContext.textFile("file:///Users/rocky/data/infos.txt")

    //注意：需要导入隐式转换
    import spark.implicits._
    val infoDF = rdd.map(_.split(",")).map(line => Info(line(0).toInt, line(1), line(2).toInt)).toDF()

    infoDF.show()

    infoDF.filter(infoDF.col("age") > 30).show

    infoDF.createOrReplaceTempView("infos")
    spark.sql("select * from infos where age > 30").show()
  }

  case class Info(id: Int, name: String, age: Int)
    
    
  def program(spark: SparkSession): Unit = {
    // RDD ==> DataFrame第二种方式
    val rdd = spark.sparkContext.textFile("file:///Users/rocky/data/infos.txt")

    val infoRDD = rdd.map(_.split(",")).map(line => Row(line(0).toInt, line(1), line(2).toInt))

    val structType = StructType(Array(StructField("id", IntegerType, true),
      StructField("name", StringType, true),
      StructField("age", IntegerType, true)))

    val infoDF = spark.createDataFrame(infoRDD,structType)
    infoDF.printSchema()
    infoDF.show()


    //通过df的api进行操作
    infoDF.filter(infoDF.col("age") > 30).show

    //通过sql的方式进行操作
    infoDF.createOrReplaceTempView("infos")
    spark.sql("select * from infos where age > 30").show()
  }


}
```

### DataSet

1.  解析csv文件

    ```scala
    def main(args: Array[String]) {
        val spark = SparkSession.builder().appName("DatasetApp")
        .master("local[2]").getOrCreate()
    
        //注意：需要导入隐式转换
        import spark.implicits._
    
        val path = "file:///Users/rocky/data/sales.csv"
    
        //spark如何解析csv文件？
        val df = spark.read.option("header","true").option("inferSchema","true").csv(path)
        df.show
    	//dataFrame => dataSet
        val ds = df.as[Sales]
        ds.map(line => line.itemId).show
    
    
        spark.sql("seletc name from person").show
    
        //df.seletc("name")
        df.select("nname")
    
        ds.map(line => line.itemId)
    
        spark.stop()
    }
    
    case class Sales(transactionId:Int,customerId:Int,itemId:Int,amountPaid:Double)
    ```

2.  Dataset是强类型

3.  DataSet在字段名错误时会报错, DataFrame和SQL不会

## 外部数据源

### 读写操作:

```scala
spark.read.format(foramtFile).load(path)

spark.write.format(foramtFile).save(path)
```

### 处理parquet(列式存储格式)

1.  spark处理Parquet文件采用的是 path/to/table可以使用文件树访问文件列

    >   ```text
    >   path
    >   └── to
    >       └── table
    >           ├── gender=male
    >           │   ├── ...
    >           │   │
    >           │   ├── country=US
    >           │   │   └── data.parquet
    >           │   ├── country=CN
    >           │   │   └── data.parquet
    >           │   └── ...
    >           └── gender=female
    >               ├── ...
    >               │
    >               ├── country=US
    >               │   └── data.parquet
    >               ├── country=CN
    >               │   └── data.parquet
    >               └── ...
    >   ```

```scala
val spark = SparkSession.builder().appName("SparkSessionApp")
.master("local[2]").getOrCreate()


/**
 * spark.read.format("parquet").load 这是标准写法
 */
val userDF = spark.read.format("parquet").load("file:///home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/src/main/resources/users.parquet")

userDF.printSchema()
userDF.show()

userDF.select("name","favorite_color").show

userDF.select("name","favorite_color").write.format("json").save("file:///home/hadoop/tmp/jsonout")

spark.read.load("file:///home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/src/main/resources/users.parquet").show

//会报错，因为sparksql默认处理的format就是parquet
spark.read.load("file:///home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/src/main/resources/people.json").show

spark.read.format("parquet").option("path","file:///home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/src/main/resources/users.parquet").load().show
spark.stop()
```

### 操作hive表和mysql表

```scala
def main(args: Array[String]) {
    val spark = SparkSession.builder().appName("HiveMySQLApp")
    .master("local[2]").getOrCreate()

    // 加载Hive表数据(需要先将hive-site.xml放入/conf下)
    val hiveDF = spark.table("emp")

    // 加载MySQL表数据
    val mysqlDF = spark.read.format("jdbc").option("url", "jdbc:mysql://localhost:3306").option("dbtable", "spark.DEPT").option("user", "root").option("password", "root").option("driver", "com.mysql.jdbc.Driver").load()

    // JOIN
    val resultDF = hiveDF.join(mysqlDF, hiveDF.col("deptno") === mysqlDF.col("DEPTNO"))
    resultDF.show


    resultDF.select(hiveDF.col("empno"),hiveDF.col("ename"),
                    mysqlDF.col("deptno"), mysqlDF.col("dname")).show

    spark.stop()
}
```



### DataFrame数据存入关系型数据库中

```scala
try {
    videoAccessTopNStatDF.foreachPartition(partitionOfRecords => {
        val list = new ListBuffer[DayVideoAccessStat]

        partitionOfRecords.foreach(info => {
            val day = info.getAs[String]("day")
            val cmsId = info.getAs[Long]("cmsId")
            val times = info.getAs[Long]("times")

            list.append(DayVideoAccessStat(day, cmsId, times))
        })

        statDAO.insertDayVideoAccessTopN(list)

    })
} catch {
    case exception: Exception => exception.printStackTrace()
}
//其中DayVideoAccessStat(day, cmsId, times)为自己所写操作数据库函数
```



## Spark愿景

1.  写更少代码
2.  读更少数据
3.  让优化器做更多的工作

## 日志清洗

调优点：
1) 控制文件输出的大小： coalesce
2) 分区字段的数据类型调整：spark.sql.sources.partitionColumnTypeInference.enabled
3) 批量插入数据库数据，提交使用batch操作

### 常见的可视化框架

1）echarts
2）highcharts
3）D3.js
4）HUE 
5）Zeppelin



## 运行模式

在Spark中，支持4种运行模式：
1）Local：开发时使用
2）Standalone： 是Spark自带的，如果一个集群是Standalone的话，那么就需要在多台机器上同时部署Spark环境
3）YARN：建议大家在生产上使用该模式，统一使用YARN进行整个集群作业(MR、Spark)的资源调度
4）Mesos

不管使用什么模式，Spark应用程序的代码是一模一样的，只需要在提交的时候通过--master参数来指定我们的运行模式即可

**Client**
	Driver运行在Client端(提交Spark作业的机器)
	Client会和请求到的Container进行通信来完成作业的调度和执行，Client是不能退出的
	日志信息会在控制台输出：便于我们测试

**Cluster**
	Driver运行在ApplicationMaster中
	Client只要提交完作业之后就可以关掉，因为作业已经在YARN上运行了
	日志是在终端看不到的，因为日志是在Driver上，只能通过yarn logs -applicationIdapplication_id



**spark提交jar包运行**

```shell
./bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn-cluster \
--executor-memory 1G \
--num-executors 1 \
/home/hadoop/app/spark-2.1.0-bin-2.6.0-cdh5.7.0/examples/jars/spark-examples_2.11-2.1.0.jar \
4
```

>   此处的yarn就是我们的yarn client模式
>   如果是yarn cluster模式的话，yarn-cluster



如果想运行在YARN之上，那么就必须要设置HADOOP_CONF_DIR或者是YARN_CONF_DIR

1） export HADOOP_CONF_DIR=/home/hadoop/app/hadoop-2.6.0-cdh5.7.0/etc/hadoop
2) $SPARK_HOME/conf/spark-env.sh 添加上面那句话



**spark打包jar包时, 需要在pom.xml中添加**

```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <mainClass></mainClass>
            </manifest>
        </archive>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
    </configuration>
</plugin>
```





## 性能调优

>存储格式的选择：http://www.infoq.com/cn/articles/bigdata-store-choose/
>压缩格式的选择：https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-compression-analysis/





调整并行度

```shell
./bin/spark-submit \
--class com.imooc.log.TopNStatJobYARN \
--name TopNStatJobYARN \
--master yarn \
--executor-memory 1G \
--num-executors 1 \
--conf spark.sql.shuffle.partitions=100 \  #并行度调整
/home/hadoop/lib/sql-1.0-jar-with-dependencies.jar \
hdfs://hadoop001:8020/imooc/clean 20170511 
```



## Spark SQL 务必要掌握的N件事情

1.  **Spark SQL使用场景**
    1.  对流数据使用SQL进行实时分析
    2.  Ad-hoc(即席) 查询
        1.  用户根据自己的需求，灵活的选择查询条件，系统能够根据用户的选择生成相应的统计报表。即席查询与普通应用查询最大的不同是普通的应用查询是定制开发的，而即席查询是由用户自定义查询条件的。
    3.  ETL处理
        1.  描述将数据从来源端经过抽取（extract）、转换（transform）、加载（load）至目的端的过程。
    4.  外部数据源交互
    5.  大集群中能够扩展查询的性能
2.  **加载数据**
    1.  加载数据到DataFrame中
    2.  加载数据到RDD中并且转换他
    3.  从云端或者本地加载数据
3.  **DataFrame vs SQL**
    1.  DataFrame = RDD + Schema
    2.  DataFrame是行式数据集的一种别名
    3.  DataFrame对于RDD: 催化剂 优化和schema
    4.  DataFrame能处理 Text, JSON, Parquet还有更多文件
    5.  SQL和API在DataFrame中都在底层进行过了优化
4.  **Schema**
    1.  隐式(inferred)
    2.  显示
5.  **SaveMode**
    1.  SaveMode.ErrorIfExists(default)
    2.  SaveMode.Append 追加
    3.  SaveMode.Overwrite 覆盖
    4.  SaveMode.Ignore  This is similar to a `CREATE TABLE IF NOT EXISTS` in SQL



