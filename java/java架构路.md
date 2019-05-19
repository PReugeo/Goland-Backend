## java架构路



### 新手

java基础使用 thinking in java入门，适合小白和初级人员

因为该书比较老（相比顶级峰会和论文）所以不适合专家

1. java基础
    1. 面向对象。（封装，继承，多态）概念
    2. java语言基础:jvm概念,半静态,.class定义,环境变量,操作符,基本语法(if,for),内置变量类型等.
    3. **jvm基础**: 内存概念， 初始化过程， gc过程， 访问控制， 变量类型
    4. **java-api**： java容器系列（list, map）,string, io,泛型等
    5. **简单设计**：方法重写、重载，java类继承，简单设计方法等



问题：

Q1:static一个方法和static一个变量,使用他们的时候访问的内存是一样的么?
Q2:方法重写时,需要不需要方法的返回值必须相同.

HashMap 看一看



### 初级

1. jdbc.事实上软件的本质也是信息和数据的处理.所以作为java访问数据库的基础.jdbc肯定肯定是需要完全了解的.
2. ORM框架,目前比较火的应该还是mybatis.这些半orm框架是一定要掌握的.可以学习helloword然后思考功能实现,最后思考数据层如何设计
3. spring.spring作为一个框架,现在基本上已经脱离框架了.现在更像一个完备的技术栈.ps.我完全没有学习过springboot.因为我学习的时候还没有springboot.现在也完全不用springboot.对于新人来说**必须**先学习spring的基础之后再来学习其他的
4. 服务端mvc框架,推荐springmvc.如果是cs类的架构可能会用一些restfull的框架,不过无所谓.这些架构还比较简单.
5. web容器.tomcat了解一下?



Q1:简单介绍一下服务器使用tomcat情况下类加载器和spring类加载器的交互
Q2:为什么说mybatic是半orm框架