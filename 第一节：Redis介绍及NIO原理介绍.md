[TOC]

# 前导

![101](8A27C1811B9A4136A387F7FD31FBD707)

## 常识：指标：寻址、带宽

    磁盘有两个指标：1.寻址(以ms为单位)；2.带宽（G/M）
    内存的两个指标：1.寻址（是ns级的）；2.带宽很大
    秒>毫秒>微秒>纳秒  磁盘比内存在寻址上慢了10W倍
    
    I/O buffer：涉及成本问题
    磁盘与磁道，扇区，一扇区(512Byte); 1T硬盘的话，带来一个成本变大：索引
    格式化以4K对齐  操作系统，无论你读多少，都是最少4k从磁盘拿

## 文件，读数据

假设文件data.txt里有数据,如果读到数据呢?linux可以使用命令grep、awk，java可以使用io流来读取数据

随着文件变大，速度变慢，为什么呢？ 磁盘I/O成为瓶颈

## 这时候就有了数据库

    **数据库中有data page的概念，data page的大小为4K，这样就跟磁盘的4K对应起来了**
    
    **data page如果定义为1K，有什么影响呢？因为从磁盘读取的时候，读取的大小为4K，造成了IO流浪费；**
    
    如果定义比4K大有什么影响？没有影响
    
    **其实磁盘读取数据大小，取决于你上层应用对IO的使用量，是IO密集型呢，还是一些小文件；如果是视频(也就是IO密集型)，那么data page可以设大，减少磁头寻址的时间**
    
    如果数据库中只建了表，没有建索引，速度还是很慢，想提升速度就需要索引
    
    **索引也是采用的4K模型**
    
    关系型数据库建表：必须给出schema(schema:一共有多少列，每个列的类型，=字节宽度)；数据库存储的时候，倾向于行级存储（便于增删改；如果某列字段为空，会用0来填充）
    
    **数据和索引都是存储在硬盘当中的，真正查询的时候，在内存中准备一个B+树（树干是在内存中的）**
    
    查询的where条件命中索引了，B+树就会走树干，从而找到叶子；也就是先将索引加载到内存，再将data page数据加载到内存中
    
==‘B+树的树干在内存中，索引和数据在磁盘’这样设计的好处就是：充分利用各自的能力，磁盘能存很多数据，内存速度快，又有一个数据结构可以加快你查找的速度，数据又是分而治之的存储，所以获取数据的速度极其快，最终的目的就是减少IO的流量==
    
## 面试题：数据库的表很大，性能下降，这句话描述对不对？

    回答这个问题，要回答两个指标：寻址、带宽
    如果表有索引
    增删改 变慢（维护索引会让增删改 变慢）
    查询速度呢？
    1，1个或少量查询依然很快（寻址；一个where条件过来，还是走内存B+树，还是一个索引块，索引到内存，data page到内存）
    2，并发大的时候，速度会受 硬盘带宽 的影响（带宽）

## 内存级别的数据库
    
SAP公司有一个HANA数据库，是一个内存级别的关系型数据库，内存2T

这里有一个常识：==数据在磁盘和内存 体积不一样==（**磁盘没有指针的概念，对一个数据的缩引都会记录在索引中**，所以==数据涨出==，==在内存中体积要小一些==）

## 折中方案：缓存

内存级别的数据库费用太贵；而存储在磁盘的数据库速度又很慢，这时候就出现了折中方案：缓存，memcached、redis

==为什么会出现缓存，究其原因就是因为2个基础设施：1.冯诺依曼体系的硬件；2.以太网，TCP/IP的网络==

## 补知识点：https://db-engines.com/en/

做为一名架构师，一定要会技术选型，也要会做技术对比；

那么怎么做呢，可以访问https://db-engines.com/en/

另外呢将来要做==项目标书==，那么涉及到的技术就可以直接在Systems中找到相关的内容，

里面有DB-Engines排名、Systems

Systems中我们先看mysql、 redis

Mysql

    数据库模型：关系型数据库
    最早版本：1995
    最新版本:8.0.21, 2020
    SQL:yes
    备份：Multi-source、Source-replica
    
Redis

    数据库模型：K-V
    最早版本：2009
    最新版本：6.0.6, July 2020
    SQL:no
    支持的语言：很多
    集群模式：sharding增片
    备份：Multi-source、Source-replica
    Redis标签：1.5M ops/sec 就是15万次操作，也就是人们常说的十万次级
        而磁盘型关系数据库，千次级，中间的性能差距就很明显了
    
# Redis

![102](438F9B648E3542749515E2289C661362)

> redis的学习网站：
> 1.  redis.cn
> 2.  redis.io
> 3.  db-engines.com

官网的介绍：

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。

它支持多种类型的数据结构，如==字符串(strings)， 散列（hashes），列表（lists）， 集合（sets）， 有序集合（sorted sets)== 与范围查询， ==bitmaps==， hyperloglogs 和 地理空间（geospatial）索引半径查询。 

**Redis 内置了 复制（replication），LUA脚本（Lua scripting），LRU驱动事件（LRU eviction），事务（transactions）和不同级别的磁盘持久化（persistence），并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。**

---

==一定要清楚:支持的5大数据类型，还有String引出的  字符类型、数值类型、bitmaps==

## 与Memcached对比

在redis之前就已经存在Memcached,也是key-value的数据结构，为什么Memcached被抛弃，就是因为==value没有类型的概念==

看到没有类型这个词，条件反射就是json

    json
    表示很复杂的数据结构
    世界上有3中数据表示
    k=a  k=1  单元素
    k=[1,2,3]   k=[a,x,f]   线性元素
    k={x=y} k=[{},{}]   k-v  数组包含对象

现在看起来value有没有类型也无所谓，memcached可以通过json来实现各种数据类型

==那么Redis中Value支持各种数据类型的意义在哪里？==

> 如果一个客户端，要从K-V类型的缓存中取回V中某一个元素，
> 
> Memcached：1.需要返回value所有的数据到客户端；2.服务器的网卡IO会增加；3.客户端需要实现代码去解码
> 
> 而Redis呢，首先呢要明白类型不是最重要的，重要的是Redis的Server对每种数据类型都有自己的方法，例如：index()  lpop();规避了Memcached存在的弊端
> 
> 也就是大数据中常说的，==计算向数据移动==

## Redis安装

环境准备：Centos 6.X   redis 5.x（安装官网最新即可）

    1.yum install wget   //有了这个命令才能下载源码
    2.cd ~
    3.mkdir soft
    4.cd soft
    5.wget    http://download.redis.io/releases/redis-5.0.5.tar.gz
    6.tar xf    redis...tar.gz  //**小知识：不需要-v，规避IO；因为通过加了v会返回数据包到客户端**
    7.cd redis-5.0.5
    8.vi README.md   //任何源码的压缩包，上来先看readme
    
    //make是编译工具，是linux系统带的;make不知道不同的源码包怎么编译，所以make 需要跟随Makefile文件；nginx 默认没有Makefile，执行了config以后就有Makefile文件了
    9.make ....yum install  gcc  
     ....  make distclean   //清除编译的文件
    
    vi Makefile //可以看到执行脚本，在src上执行Make命令
    10.make
    11.cd src   ....生成了可执行程序
    vi Makefile   //这个才是真正的Makefile文件，搜索PREFIX(默认为/usr/local)、install
    
    12.cd ..
    13.make install PREFIX=/opt/mashibing/redis5   //提示没有cc命令的话,说明缺少编译环境，需要yum install gcc
    //如果安装失败，可能是以前安装失败，需要执行make distclean
    //make成功以后，在src下就有可执行程序了，例如redis-server、redis-cli
    
    make test   //可以不执行，企业环境建议执行；编译成功以后的测试
    
    cd /opt/mashibing/redis5
    cd /bin
    
    
    
    //把redis变成服务
    
    14.vi /etc/profile
    ...   export  REDIS_HOME=/opt/mashibing/redis5  //配置安装路径
    ...   export PATH=$PATH:$REDIS_HOME/bin   //将redis/bin添加到path中
    
    15.source /etc/profile    //如果不做，发现不了上面添加的配置
    
    echo $PATH  //查看path配置
    
    16.cd /soft/redis-5.0.5/utils
    17. ./install_server.sh  （可以执行一次或多次）
        //默认端口号为6379，配置位置默认为/etc/redis/6379.conf, log日志位置：/var/log/redis_6379.log,
        //数据目录（内存应用需要持久化，所以有数据目录）默认为：/var/lib/redis/6379 ，
        //可执行程序的路径为：/opt/mashibing/redis5/bin/redis-server
        //帮助做的工作：redis服务加入开机启动，服务启动
        
        a)  一个物理机中可以有多个redis实例（进程），通过port区分
        b)  可执行程序就一份在目录，但是内存中未来的多个实例需要各自的配置文件，持久化目录等资源
        c)  service   redis_6379  start/stop/stauts     >   linux   /etc/init.d/redis_6379 
        d)脚本还会帮你启动！
    
    cd /etc/init.d   //做了开机启动的话，会在/ect/init.d目录下有一个redis_6379的一个脚本，注意是脚本
    //这样就可以在任意目录执行 service redis_6379 status来查看redis服务了
    
    vi redis_6379
    
    cd 
    
    service redis_6379 status
    
    ------再按照一个redis实例
    cd /soft/redis-5.0.5/utils
    ./install_server.sh    //port:6380,配置文件地址、日志文件地址、数据地址都有6380标识
    
    service redis_6380 status
    
    
    17.ps -fe |  grep redis   //可以看到两个redis进程

## 

### 补知识：Epoll

![104](493A253B88F64C51807F4E26E587E4D7)

#### BIO：

客户端跟Kernel建立连接，就会产生一个FD,线程/进程（windows-线程；linux-进程） 读取连接中的数据时，是Blocking的，即一个连接没有数据时，读数据的这个线程/进程就会阻塞

==你知道JVM的线程成本是多少吗？==

cpu只有一颗的情况下，==JVM：一个线程的成本是1MB==，堆是共享的，线程栈是对立的,

==JVM需要注意：线程多了，1.增加调度成本(CPU浪费)；2.增加内存成本==


---

#### yum install man man-pages  

//安装man帮助程序，查看所有的man-pages;有8类文档，2类是系统调用（系统调用：内核暴露给程序的调用方法）

man 2 read  //查看read系统调用

ps -fe | grep redis

cd /proc/redis进程号/fd   //查看文件描述符

man 2 read  //查看read系统调用

man 2 socket  //type中NONBLOCK 


---

#### NIO:（同步非阻塞）

yum install man man-pages 

man 2 socket //查看socket，知道fd的type可以为NONBLOCK,非阻塞的；一个线程/进程轮询客户端的连接socket；

==注意：轮询发生在用户空间==

这个时期是==同步非阻塞==的（同步：取出连接，处理数据都由同一个线程来处理）

==如果有1000个FD，代表用户进程轮询调用1000次kernel，造成成本问题==

---


#### NIO-多路复用-select（同步非阻塞）

解决NIO的问题，内核再发展

多了select的系统调用

man 2 select //把所有的FDs传给select方法，返回可执行的FD结果，对返回的结果再调用read系统调用

==问题：用户态到内核态，FDs的相关数据需要拷贝来拷贝去，想要实现零拷贝==


---

#### NIO-多路复用-Epoll

man 2 mmap //用户态和内核态的共享空间：共享空间通过mmap系统调用实现的

man epoll  //7类杂项，==有2类的系统调用：epoll_create   epoll_ctl   epoll_wait==

关键在于：用户态和内核态的共享空间；

原先放到进程/线程的中1000个FDs,不需要再在线程中开辟空间了，放到了共享空间；

==共享空间不同于零拷贝==

==注意Epoll是NIO，不是AIO==

Linux的AIO发展历程相当复杂，只有Windows有AIO；关于Linux的AIO感兴趣自行了解

#### 零拷贝Sendfile

man 2 sendfile //零拷贝是调用系统调用sendfile(out输出，in输入)

硬盘上有一个file.txt文件，有网卡；分别创建了文件IO，Socket IO；==用户态通过系统调用read，将FD写入用户空间；再通过系统调用write将fd写到网卡==；这个过程==涉及到数据在用户态和内核态的拷贝==

有了Sendfile(),是用户调用系统调用sendfile，内核将内容读到缓存区，由缓存区直接写到网卡，没有数据在用户态和内核态之间拷贝了

sendfile 加上 mmap可以组成kafka（是基于JVM的一个用户空间进程），可以通过mmap挂载到文件files；

==通过kafka通过mmap看到的内存空间，内核也能看到，这样就减少了系统调用，减少数据拷贝,这样就能实现网卡写入文件时很快==

**消费者通过偏移量来读数据，走sendfile零拷贝**

![103](7189C6697EDC4D24BDD04EFDC446548E)

### redis是单进程单实例的；那么并发很多请求，redis如何变得很快的呢？

一台linux服务器，有两个redis：端口号分别为6379/6380；注意：==redis是单进程单实例的；那么并发很多请求，redis如何变得很快的呢？==

linux系统中有内核Kernel，客户端经过三次握手，到达Kernel，建立Socket；

Redis进程和Kernel之间使用的是Epoll（内核提供的系统调用）

内存是快的，IO是慢的

数据从客户端到达内核，是有“顺序”性的，是指每连接内的命令顺序

==这里类似Nginx，多少颗CPU启动多少个进程，就是worker进程；一个worker进程就可以压到cpu的一二三级缓存了；每个worker进程就是使用了kernel的多路复用epoll（同步非阻塞）==