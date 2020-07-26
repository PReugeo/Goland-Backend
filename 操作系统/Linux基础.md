 参考自https://github.com/CyC2018/CS-Notes 

# 常用操作命令

##  --help

指令的基本用法与选项介绍。

## man

man 是 manual 的缩写，将指令的具体信息显示出来。

当执行 `man date` 时，有 DATE(1) 出现，其中的数字代表指令的类型，常用的数字及其类型如下：

| 代号 | 类型                                            |
| ---- | ----------------------------------------------- |
| 1    | 用户在 shell 环境中可以操作的指令或者可执行文件 |
| 5    | 配置文件                                        |
| 8    | 系统管理员可以使用的管理指令                    |

## info

info 与 man 类似，但是 info 将文档分成一个个页面，每个页面可以进行跳转。



## doc

/usr/share/doc 存放着软件的一整套说明文件。



## 文件操作命令

touch   建立文件

mkdir   新建文件夹

ls    查看当前文件夹中文件

cd   进入目录

cat    读取文件内容

tail -f   从尾部读取文件

head   从头部读取

more   一行一行的读取

grep   搜索文件中内容  -n显示行数

wc   统计  -l统计行数

find   在指定目录下查找文件

sudo  使用root权限

rm   删除目录和文件

cp   复制

mv    移动

pwd 显示路径

tar   解压缩.tar  -cf创建压缩文件 -tvf  显示压缩文件中文件的详细信息

-xf   抽取文件（解压缩）

-xzvf  .gz文件解压缩

-czvf   压缩.gz文件

## 服务器资源信息查看

内存： free -m

硬盘： df -h

负载： w/top

cpu查看： cat /proc/cpuinfo

查询服务是否启动  ps -ef | grep 服务名

提权：sudo

visudo 修改权限

文件下载 wget/curl

文件上传  scp 文件名 用户名@IP:路径 

scp加入文件名后就是下载文件 

一直运行命令 nohup command &(后台运行)

## 修改时区

tzselect

cp     /usr/share/zoneinfo/Asia/Shanghai /etc/localtime



# SSH

## SSH（安全外壳协议）是什么？

是一个安全的协议，用于安全的远程连接服务器

建立在应用层基础上的安全协议

有效防止远程管理过程中信息泄露问题

服务器安装SSH服务

```shell
yum install openssh-server

Service sshd start
Chkconfig sshd on
```

设置开机运行

先在linux下安装 yum install -y lrzsz

上传文件  rz

下载文件 sz + 文件名

## SSH KEY

ssh key  会生成公钥和私钥(客户端linux)

公钥对外公开，放在服务器的~/.ssh/authorized_keys文件中

Linux 平台下使用ssh-add将私钥放进ssh中

其中私钥存在在~/.ssh目录下

Linux   平台生成ssh key方法：

ssh-keygen -t rsa/dsa  (后两个为加密算法)

## SSH config 文件

config文件存在在~/.ssh/config 没有可自行创建

可配置多个host连接同一台主机

| Host         | 别名（用于连接的别名）   |
| ------------ | ------------------------ |
| HostName     | 连接ip                   |
| User         | 使用账户                 |
| Port         | 使用端口（默认为22端口） |
| IdentityFile | 密钥文件的路径           |

客户端是mac或linux时使用



## SSH 连接

windows中：xshell，putty进行服务器

的连接

连接命令：ssh 'user'@'ip'进行连接

(所有平台皆为该命令)



# VIM

## VIM 操作模式

- 一般指令模式（Command mode）：VIM 的默认模式，可以用于移动游标查看内容；
- 编辑模式（Insert mode）：按下 "i" 等按键之后进入，可以对文本进行编辑；
- 指令列模式（Bottom-line mode）：按下 ":" 按键之后进入，用于保存退出等操作。

## VIM 快捷键

![img](D:\Java-golang-learning\java\static\vim.gif)



# 开源许可证

![img](D:\Java-golang-learning\java\static\开源许可证.png)



# 分区

## 分区表

分区表主要有两种格式，一种是 MBR（ 限制较多 ），一种是较新限制较小的 GPT 分区表。

### 1. MBR

MBR 中，第一个扇区最重要，里面有主要开机记录（Master boot record, MBR）及分区表（partition table），其中主要开机记录占 446 bytes，分区表占 64 bytes。

分区表只有 64 bytes，最多只能存储 4 个分区，这 4 个分区为主分区（Primary）和扩展分区（Extended）。其中扩展分区只有一个，它使用其它扇区来记录额外的分区表，因此通过扩展分区可以分出更多分区，这些分区称为逻辑分区。

Linux 也把分区当成文件，分区文件的命名方式为：磁盘文件名 + 编号，例如 /dev/sda1。注意，逻辑分区的编号从 5 开始

### 2. GPT

扇区是磁盘的最小存储单位，旧磁盘的扇区大小通常为 512 bytes，而最新的磁盘支持 4 k。GPT 为了兼容所有磁盘，在定义扇区上使用逻辑区块地址（Logical Block Address, LBA），LBA 默认大小为 512 bytes。

GPT 第 1 个区块记录了主要开机记录（MBR），紧接着是 33 个区块记录分区信息，并把最后的 33 个区块用于对分区信息进行备份。这 33 个区块第一个为 GPT 表头纪录，这个部份纪录了分区表本身的位置与大小和备份分区的位置，同时放置了分区表的校验码 (CRC32)，操作系统可以根据这个校验码来判断 GPT 是否正确。若有错误，可以使用备份分区进行恢复。

GPT 没有扩展分区概念，都是主分区，每个 LBA 可以分 4 个分区，因此总共可以分 4 * 32 = 128 个分区。

MBR 不支持 2.2 TB 以上的硬盘，GPT 则最多支持到 233 TB = 8 ZB。



## 开机启动程序

### BIOS

BIOS（ 基本输入输出系统 ）是一个固件，其程序存放在断电后不会丢失的只读内存中。BIOS  是开机执行的第一个程序，它自动识别可以开机的磁盘并读取磁盘的主要开机记录（ MBR ），MBR 执行其中的开机管理程序，该管理程序加载操作系统的核心文件。

MBR 有以下功能

- 选单
- 载入核心文件
- 转交其他开机管理程序

### UEFI

BIOS 不可以读取 GPT 分区表，UEFI 可以。



# 文件系统

## 分区与文件系统

对分区进行格式化是为了在分区上建立文件系统，一个分区一般只能格式化一个文件系统，但磁盘阵列等技术可以将一个分区格式化为多个文件系统。

## 组成

最主要由 inode，block 组成，除此外还有

*  superblock：记录文件系统整体信息，包括 inode 和 block 的总量，使用量、剩余量，以及文件系统格式与相关信息。
* block bitmap：记录 block 是否被使用的位图。

### inode

一个文件占用一个 inode，记录文件的属性，同时记录此文件内容所在的 block 编号；

inode 具体包含以下信息

* 权限 （ r/w/x ）
* 拥有者和群组（ owner/group ）
* 容量
* 建立或状态改变时间 （ ctime ）
* 最近读取时间（ atime ）
* 最近修改时间（ mtime ）
* 定义文件特性的旗标（ flag ），如 SetUID..
* 该文件真正内容的指针 （ pointer ）

inode 具有以下特点：

* 每个 inode 大小固定为 128 bytes（ ex4 与 xfs 可设定到 256 bytes ）
* 每个文件仅占用一个 inode

由于 inode 大小有限，无法装下大文件的所有 block 因此引入了间接、双间接、三间接引用。

### block

一个 block 只能被一个文件使用，未使用的部分会被浪费，因此如果需要存储大量小文件最好选用小一点的 block。

| 大小         | 1KB  | 2KB   | 4KB  |
| ------------ | ---- | ----- | ---- |
| 最大单一文件 | 16GB | 256GB | 2TB  |
| 最大文件系统 | 2TB  | 8TB   | 16TB |

## 目录

建立一个目录时，会分配一个 inode 和至少一个 block，block 中记录目录下所有文件的 inode 编号以及文件名。

 可以看到文件的 inode 本身不记录文件名，文件名记录在目录中，因此新增文件、删除文件、更改文件名这些操作与目录的写权限有关。 



## 目录配置

为了使不同 Linux 发行版本的目录结构保持一致性，Filesystem Hierarchy Standard (FHS) 规定了 Linux 的目录结构。最基础的三个目录如下：

- / (root, 根目录)
- /usr (unix software resource)：所有系统默认软件都会安装到这个目录；
- /var (variable)：存放系统或程序运行过程中的数据文件。

![img](D:\Java-golang-learning\操作系统\img\linux-filesystem.png)



# 链接

```shell
ln [-sf] source_filename dist_filename
-s： soft 默认是实体链接，加 -s 为符号链接
-f： force 若目标文件存在，则先删除再添加
```

## 1. 实体链接（ 硬链接 ）

在目录下创建一个条目，记录文件名和 inode 编号，这个 inode 为源文件的 inode。

删除任意一个条目，文件还是存在，只要引用数量不为 0。

 有以下限制：不能跨越文件系统、不能对目录进行链接。 

## 2. 符号链接（软链接）

符号链接保存着源文件所在的绝对路径，在读取时会定位到源文件上，类似于 Windows 的快捷方式。

当源文件被删除了，链接文件就打不开了。

因为记录的是路径，所以可以为目录建立符号链接。



# 指令与文件搜索

## 1. Which

指令搜索

```shell
which [-a] command
-a: 列出所有指令
```

## 2. Whereis

文件搜索，速度快，因为只搜索特定目录

```shell
whereis [-bmsu] dirname/filename
```

## 3. locate

文件搜索，可以使用关键字或者正则表达式进行搜索。

 locate 使用 /var/lib/mlocate/ 这个数据库来进行搜索，它存储在内存中，并且每天更新一次，所以无法用 locate 搜索新建的文件。可以使用 updatedb 来立即更新数据库。 

```shell
locate [-ir] keyword
-r: 正则表达式
```

## 4. find

文件搜索，可以使用文件的属性和权限进行搜索。

```shell
find [basedir] [option]
example: find . -name "shadow"
```

**① 与时间有关的选项**

```shell
-mtime  n ：列出在 n 天前的那一天修改过内容的文件
-mtime +n ：列出在 n 天之前 (不含 n 天本身) 修改过内容的文件
-mtime -n ：列出在 n 天之内 (含 n 天本身) 修改过内容的文件
-newer file ： 列出比 file 更新的文件Copy to clipboardErrorCopied
```

+4、4 和 -4 的指示的时间范围如下：

![img](D:\Java-golang-learning\操作系统\img\时间范围.png)



**② 与文件拥有者和所属群组有关的选项**

```shell
-uid n
-gid n
-user name
-group name
-nouser ：搜索拥有者不存在 /etc/passwd 的文件
-nogroup：搜索所属群组不存在于 /etc/group 的文件Copy to clipboardErrorCopied
```

**③ 与文件权限和名称有关的选项**

```shell
-name filename
-size [+-]SIZE：搜寻比 SIZE 还要大 (+) 或小 (-) 的文件。这个 SIZE 的规格有：c: 代表 byte，k: 代表 1024bytes。所以，要找比 50KB 还要大的文件，就是 -size +50k
-type TYPE
-perm mode  ：搜索权限等于 mode 的文件
-perm -mode ：搜索权限包含 mode 的文件
-perm /mode ：搜索权限包含任一 mode 的文件
```

# 进程管理

## 查看进程

1. ps 查看某个时间点的进程信息。

```shell
ps -l  #查看自己的进程
ps aux #查看系统所有进程
ps aux | grep xxx #查看特定进程
```

2. pstree 查看进程树

```shell
pstree -A #查看所有进程树
```

3. top 实时显示进程信息

```shell
top -d 2 #每 2 秒刷新一次
```

4. netstat 查看占用端口的进程

```shell
netstat -anp | grep port #查看特定端口的进程
```

## 进程状态

| 状态 | 说明                                                         |
| :--: | :----------------------------------------------------------- |
|  R   | running or runnable (on run queue) 正在执行或者可执行，此时进程位于执行队列中。 |
|  D   | uninterruptible sleep (usually I/O) 不可中断阻塞，通常为 IO 阻塞。 |
|  S   | interruptible sleep (waiting for an event to complete) 可中断阻塞，此时进程正在等待某个事件完成。 |
|  Z   | zombie (terminated but not reaped by its parent) 僵死，进程已经终止但是尚未被其父进程获取信息。 |
|  T   | stopped (either by a job control signal or because it is being traced) 结束，进程既可以被作业控制信号结束，也可能是正在被追踪 |

![img](..\操作系统\img\进程图.png)



## SIGCHLD

当一个子进程改变了它的状态时（停止运行，继续运行或者退出），有两件事会发生在父进程中：

- 得到 SIGCHLD 信号；
- waitpid() 或者 wait() 调用会返回。

其中子进程发送的 SIGCHLD 信号包含子进程的信息，比如进程 ID、进程状态、进程使用 CPU 的时间等。

子进程退出时，它的进程描述不会立即释放，是为了让父进程得到子进程信息，父进程通过 wait() 和 waitpid() 来获得一个退出的子进程的信息。

## 孤儿进程

一个父进程退出，而它的一个或多个子进程还在运行，那么这些子进程将成为孤儿进程。

孤儿进程将被 init 进程（进程号为 1）所收养，并由 init 进程对它们完成状态收集工作。

由于孤儿进程会被 init 进程收养，所以孤儿进程不会对系统造成危害。

## 僵尸进程

一个子进程的进程描述符在子进程退出时不会释放，只有当父进程通过 wait() 或 waitpid() 获取了子进程信息后才会释放。如果子进程退出，而父进程并没有调用 wait() 或 waitpid()，那么子进程的进程描述符仍然保存在系统中，这种进程称之为僵尸进程。

僵尸进程通过 ps 命令显示出来的状态为 Z（zombie）。

系统所能使用的进程号是有限的，如果产生大量僵尸进程，将因为没有可用的进程号而导致系统不能产生新的进程。

要消灭系统中大量的僵尸进程，只需要将其父进程杀死，此时僵尸进程就会变成孤儿进程，从而被 init 进程所收养，这样 init 进程就会释放所有的僵尸进程所占有的资源，从而结束僵尸进程。

