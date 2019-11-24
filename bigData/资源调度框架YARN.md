# 资源调度框架YARN

### 作用

不同计算框架可以共享同一个HDFS集群上的数据, 享受整体的资源调度

通用的资源管理系统

为上层应用提供统一资源调度



### 架构

**1)  ResourceManager: RM**

​	整个集群同一时间提供服务的RM只有一个, 负责集群资源的统一管理和调度

​	处理客户端的请求

​	监视NM, 一旦某个Nm挂了, 那么该NM上运行的任务需要告诉AM来如何进行处理

**2) NodeManager: NM**

​	整个集群中有多个, 负责自己本身节点资源管理和使用

​	定时向RM汇报本节点资源使用情况

​	接受并处理来自RM的各种命令:启动Container

​	处理来自AM的命令

​	单个节点资源管理由自己管理

**3) ApplicationMaster: AM**

​	每个应用程序对应一个:MR, Spark, 负责应用程序的管理

​	为应用程序向RM申请资源(core, memory), 分配给内部task

​	需要与NM通信:启动／停止task， task是运行在Container里面， AM也在Container里面

**4) Container**

​	封装了CPU、Memory等资源的一个容器

​	是一个任务运行环境的抽象

**5) Client**

​	提交作业

​	查询作业的进度

​	杀死作业

​	

```xml
<!-- yarn-site.xml -->

<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>localhost</value>
    </property>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

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
<!-- Site specific YARN configuration properties -->


```

### YARN优点

可扩展性(Scalability)

可用性强

利用率高

多租户