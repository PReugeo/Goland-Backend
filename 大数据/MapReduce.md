# 分布式处理框架MapReduce

### 概述

*   Hadoop MapReduce是Google MapReduce的克隆版
*   优点: 海量数据离线处理&易开发&易运行
*   缺点: 无法实时流式计算

### MapReduce过程概览

![](D:\Java-golang-learning\bigData\MapReduce.assets/20181203204646963.png)

### 核心概念

*   Spilit: 交由MapReduce作业来处理的数据库, 是MapReduce中最小的计算单元

    一般HDFS: blocksize和Spilit是一一对应的

*   InputFormat:

    *   将我们的输入数据进行分片(spilit)
    *   TextInputFormat: 处理文本格式数据

*   OutputFormat: 输出

### 开发(使用IDEA+maven)

1.  开发
2.  编译: mvn clean package -DskipTests
3.  上传到服务器
4.  运行: hadoop jar xxx.jar 主类



在MapReduce中, 输出文件是不能事先存在的

1.  先手工通过shell的方式将输出文件夹删除(不推荐)

    1.  ```shell
        hadoop fs -rm -r /output
        ```

2.  在代码中完成自动删除功能(推荐)

3.  ```java
    //准备清理已存在的输出文件夹
    Path outputPath = new Path(args[1]);
    FileSystem fileSystem = FileSystem.get(configuration);
    if (fileSystem.exists(outputPath)) {
        fileSystem.delete(outputPath, true);
        System.out.println("output file exists but it already deleted");
    }
    ```

    

#### Combiner

1.  本地的reducer

2.  减少Map Tasks输出的数据量及数据网络传输量

3.  例子

    1.  ```java
        //设置map参数
        job.setMapperClass(MyMapper.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);
        
        //设置reduce参数
        job.setReducerClass(MyReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);
        
        //通过job设置combiner处理类, 逻辑上和reduce一样
        job.setCombinerClass(MyReducer.class);
        ```

4.  适用场景: 求和、次数， 不适用于求平均数



#### Partitioner

1.  它决定MapTask输出的数据交由哪个ReduceTask处理

2.  默认实现： 分发的key的hash值对Reduce Task个数取模

3.  ```java
        public static class MyPartitioner extends Partitioner<Text, LongWritable> {
            @Override
            public int getPartition(@NotNull Text key, LongWritable value, int numPartitions) {
                if ("xiaomi".equals(key.toString())) {
                    return 0;
                }
    
                if ("huawei".equals(key.toString())) {
                    return 1;
                }
    
                if ("iphone".equals(key.toString())) {
                    return 2;
                }
    
                return 3;
            }
        }
    
            //通过job设置Partitioner
            job.setPartitionerClass(MyPartitioner.class);
            //设置4个reducer， 每个分区一个
            job.setNumReduceTasks(4);
    ```
  ```



####  JobHistory

记录已运行完的MapReduce信息到指定的HDFS目录下

19888端口默认不开启

​```xml
    <!--jobhistory启动配置 mapred-site.xml-->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>localhost:10020</value>
        <description>MapReduce JobHistory Server IPC host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>localhost:19888</value>
        <description>MapReduce JobHistory Server Web UI host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/history/done</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/history/done_intermediate</value>
    </property>

    <!--jobhistory聚合开启配置 yarn-site.xml-->
    <!-- yarn开启聚合 -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>302400</value>
	    <description>
		     日志文件保存在文件系统（如HDFS文件系统）的最长时间，默认值是-1，即永久有效。
		     这里配置的值是：7天 = 3600 * 24 * 7 = 302400
		</description>
    </property>
  ```



### MapReduce的局限性

1.  代码繁琐
2.  只支持map和reduce方法
3.  执行效率低下
4.  不适合迭代多次,交互式,流失的处理

## WordCount

```java
	public void map(Object key, Text value, Context context) throws IOException,InterruptedException {
		StringTokenizer itr = new StringTokenizer(value.toString());
		while(itr.hasMoreTokens()) {
			word.set(itr.nextToken());
			context.write(word, one);
		}
	}
	public void reduce(Text	key, Iterable<IntWritable> values, Context context) throws IOException,InterruptedException {
		int sum = 0;
		for(IntWritable val:values) {
			sum += val.get();
		}
		result.set(sum);
		context.write(key,result);
	}
```

