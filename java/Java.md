# Java基础

## 数据类型

### 自动装箱

```java
Integer x = 2;     // 装箱 调用了 Integer.valueOf(2)
int y = x;         // 拆箱 调用了 X.intValue()
```

### 缓存池

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

- new Integer(123) 每次都会新建一个对象；
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

自动装箱过程中会自动调用 valueOf()

> Java8 中基本类型对应的缓冲池如下
>
> - boolean values true and false
> - all byte values
> - short values between -128 and 127
> - int values between -128 and 127
> - char in the range \u0000 to \u007F

### String

String 被声明为 final ,因此它不可被继承。且不可变

```java
//Java8 中 String 声明
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char value[];
}

//Java9 中改用 byte 数组存储，同时使用 coder 表明使用哪种编码
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}

```

不可变好处： 

1. 线程安全
2. 安全性好
3. 可以缓存 hash 值
4. String pool 的需要

#### String， StringBuffer， StringBuilder 比较

1. String， StringBuffer 线程安全 --》 StringBuffer 加了同步锁
2. StringBuffer， StringBuilder 为可变字符串
3. StringBuilder 性能稍微比 StringBuffer 好

#### String Pool

 字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程中将字符串添加到 String Pool 中。 若 String Pool 存在该字符串值相等的则直接返回字符串池的地址 equals() 方法确定。String Pool 在堆上，若大量使用会报 OutOfMemoryError

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
String s4 = s1.intern();
System.out.println(s3 == s4);           // true
//如果使用 String str = "aaa" 则会自动将其加入字符串常量池中
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```

## 运算

### 参数传递

Java 的参数是使用值传递而非引用传递

基本类型使用 = 赋值，值存在变量中

引用类型（对象）使用 = 赋值，实际对象的地址保存在变量中

只有使用对象自有的方法改变其地址指向才能改变值

### 隐式类型转换

Java 不能隐式向下转型（精度高 -> 精度低）

> +=，-=，++， *=， /= 等赋值符可以进行向下转型

## 继承

### 访问权限

Java 中有三个访问权限修饰符：public，private，protected，不加修饰符为 default 

![img](D:\Java-golang-learning\java\static\修饰符权限.png)

### 抽象类与接口

抽象类使用 abstract 关键字声明。抽象类中不一定有抽象方法，但有抽象方法一定是抽象类。

抽象类不能被实例化，只能被继承，子类必须也是抽象类或者实现了父类的抽象方法。

```java
public abstract class AbstractClassExample {

    protected int x;
    private int y;

    public abstract void func1();

    public void func2() {
        System.out.println("func2");
    }
}
public class AbstractExtendClassExample extends AbstractClassExample {
    @Override
    public void func1() {
        System.out.println("func1");
    }
}
// AbstractClassExample ac1 = new AbstractClassExample(); // 'AbstractClassExample' is abstract; cannot be instantiated
AbstractClassExample ac2 = new AbstractExtendClassExample();
ac2.func1();
```

接口在 Java8 之前可以看成完全抽象的类。Java8 后可以有默认实现。

接口的成员（字段 + 方法）默认都是 public 的，并且不允许定义为 private 或者 protected。

接口的字段默认都是 static 和 final 的。



抽象类与接口的区别？

- 从设计层面上看，抽象类提供了一种 IS-A 关系，那么就必须满足里式替换原则，即子类对象必须能够替换掉所有父类对象。而接口更像是一种 LIKE-A 关系，它只是提供一种方法实现契约，并不要求接口和实现接口的类具有 IS-A 关系。
- 从使用上来看，一个类可以实现多个接口，但是不能继承多个抽象类。
- 接口的字段只能是 static 和 final 类型的，而抽象类的字段没有这种限制。
- 接口的成员只能是 public 的，而抽象类的成员可以有多种访问权限。

使用上，接口优先级大部分时间优先于抽象类。

### super

1. super 用于访问父类的构造函数，若子类未使用 super 则默认调用父类无参构造函数。所以 Java 类需要定义无参数不做事的构造函数，防止编译错误。
2. super 用于访问父类的成员（protected，public）。也可以访问父类的方法

### 重载和重写

重写为子类重写父类方法

重载为同一个方法名，但参数类型或个数或顺序不同



## Object 类通用方法

```java
toString(); //返回对象的字符串表示
hashCode();	//返回该对象的哈希码值
clone();	//克隆对象
equals();	//值比较值是否相等，引用比较地址是否相等，通常重写
finalize();	//被垃圾回收器对象调用，用于回收垃圾
getClass();	//返回此 Object 运行的类
```

> == 和 equals() 的区别？
>
> ​	== 可以比较基本类型的值是否相同， equals 只能比较对象的地址
>
> ​	equals 可能被重写

hashcode() 和 equals() ：

1. hashcode 用于确定对象在散列表中的索引位置，大大减少 equals 使用频率。先比较 hashcode 后使用 equals 比较
2. 两对象相等则 hashcode 相同，反之则不一定
3. equals 重写，hashcode 也要重写
4. hashcode 默认为对堆上的对象产生独特值，若 hashcode 没有重写，则该 class 下的对象不会相等。

## 关键字

### final

- 修饰类时:类不能被继承
- 修饰变量时: 变量变成了常量,只能被复制一次
- 修饰方法:方法不能被重写

### static

1. 静态变量

    1. 静态变量又称为类变量，类中所有实例共享该变量，一个类中只存在一份

    2. 实例变量：每创建一个实例就产生一个实例变量。

    3. ```java
        public class A {
            private int x; 		  //实例变量
            private static int y; //静态变量
        }
        ```

2. 静态方法

    1. 静态方法不依赖实例（通过类名使用），必须有实现，不能为抽象方法。
    2. 只能访问静态字段和静态方法，方法中不能有 this 和 super。

3. 静态代码块

    1. 比构造代码块和构造方法更早执行，在类初始化时运行一次。

4. 内部静态类

    1. 非静态内部类依赖外部类实例来访问，内部静态类不需要

    2. ```java
        public class OuterClass {
        
            class InnerClass {
            }
        
            static class StaticInnerClass {
            }
        
            public static void main(String[] args) {
                // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
                OuterClass outerClass = new OuterClass();
                InnerClass innerClass = outerClass.new InnerClass();
                StaticInnerClass staticInnerClass = new StaticInnerClass();
            }
        }
        
        ```

    3. 不能访问非静态变量和方法

5. 静态导包，  不用再指明 ClassName，从而简化代码，但可读性大大降低。 

6. 初始化顺序

    1. 父类（静态变量、静态语句块）
    2. 子类（静态变量、静态语句块）
    3. 父类（实例变量、普通语句块）
    4. 父类（构造函数）
    5. 子类（实例变量、普通语句块）
    6. 子类（构造函数）

## 反射

反射可以提供运行时的类的信息，而且是运行时才加载进来。

Class 和 java.lang.reflect 一起对反射提供支持， reflect 类库包含以下三个类

- **Field** ：可以使用 get() 和 set() 方法读取和修改 Field 对象关联的字段；
- **Method** ：可以使用 invoke() 方法调用与 Method 对象关联的方法；
- **Constructor** ：可以用 Constructor 的 newInstance() 创建新的对象。

**反射的优点：**

- **可扩展性** ：应用程序可以利用全限定名创建可扩展对象的实例，来使用来自外部的用户自定义类。
- **类浏览器和可视化开发环境** ：一个类浏览器需要可以枚举类的成员。可视化开发环境（如 IDE）可以从利用反射中可用的类型信息中受益，以帮助程序员编写正确的代码。
- **调试器和测试工具** ： 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API 定义，以确保一组测试中有较高的代码覆盖率。

**反射的缺点：**

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

- **性能开销** ：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
- **安全限制** ：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。
- **内部暴露** ：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

## 异常

Throwable 可以用来表示任何可以作为异常抛出的类，分为两种： **Error** 和 **Exception**。其中 Error 用来表示 JVM 无法处理的错误，Exception 分为两种：

- **受检异常** ：需要用 try...catch... 语句捕获并进行处理，并且可以从异常中恢复；
- **非受检异常** ：是程序运行时错误，例如除 0 会引发 Arithmetic Exception，此时程序崩溃并且无法恢复。

## 泛型

(1)泛型：是一种把明确数据类型的工作推迟到创建对象或者调用方法的时候采取明确的特殊的数据类型。

泛型是通过类型擦除实现的，编译器在编译时擦除了所有类型的相关信息。

目的是兼容 Java5 之前开发二进制类库。

好处：

A:提高了程序的安全性

B:把运行时期异常提前到了编译期

C:避免了强制类型转换



泛型通配符<?>

   任意类型，如果没有明确，那么就是Object以及任意的Java类了

extends E

  向下限定，E及其子类

super E

​     向上限定，E及其父类