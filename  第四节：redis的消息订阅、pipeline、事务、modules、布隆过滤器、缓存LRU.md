[TOC]

# 回顾

之前讲了Redis的安装、五种类型的命令行使用

# Redis的进阶使用

可以通过Redis官网进行学习

redis服务启不起来的排查过程
    
    service redis_6379 start //启动
    service redis_6379 status //查看状态
    rm -fr /var/run/redis_6379.pid
    
    service redis_6379 start //再次启动
    service redis_6379 status //再次查看状态
    
    cd  /var/lib/redis
    cd 6379
    rm -fr ./*
    
    vi /etc/redis/6379.conf  //查看配置文件
    
    redis-server /ect/redis/6379.conf
    
    ps -fe | grep redis  //发现redis启动了
    


只要跟redis建立连接，给redis发送指令，它就能执行；通信协议很简单

    redis-cli  //连接redis客户端
    keys *  //空的
    
    yum install nc //安装navicat
    
    nc localhost 6379  //连接redis
    
        keys * 
        set k1 hello
        
    keys *  //有k1了
    
## 管道pipeline

如果想要发送更多的数据，可以使用管道

管道让我们通信的成本变低

    echo -e "sdfsdf\nsdfsdf"  // \n 换行符   -e 可以识别换行符
    
    echo -e "set k2 99\nincr k2\n get k2" | nc localhost 6379 //建立socket连接，把多个命令发给redis
    
### redis冷启动，需要 大量插入数据 或 从文件中批量插入数据，用到管道 （了解）

**从Redis 2.6开始redis-cli支持一种新的被称之为pipe mode的新模式用于执行大量数据插入工作**。

cat data.txt | redis-cli --pipe

**redis-cli中只支持dos格式的换行符 \r\n （必须为这个格式） ，如果你在Linux下、Mac下或者Windows下创建的文件，最好都转个码。没有转码的文件,执行会失败**。****

**如果是CentOS，使用yum install unix2dos安装unix2dos转码工具。**

文件转码完成后，就可以导入，导入使用cat和redis-cli命令组合,一个用来读取文件内容,一个用来发送文件到redis执行，如果要导入的文件和redis在同一台服务器上，可以直接将本地文件中的指令导入redis执行

cat d1.txt | redis-cli

**如果你导入的指令比较多，可以使用--pipe 这个参数来启用pipe协议，它不仅仅能减少返回结果的输出，还能更快的执行指令 **

cat d1.txt | redis-cli --pipe


**作为程序员来说，一般只需要准备好大量数据即可，插入大量数据一般由运维来操作，所以这块知识了解即可**

## 发布/订阅（Pub/Sub）

之前讲到list类型的时候，提到了阻塞，单播队列 FIFO  也就是blpop(阻塞的弹出元素)时提到：

一个list可以有10 个客户端去blpop，阻塞；list到达元素后，连接拿到元素离开；是FIFO的

举个例子，直播，想要实现快速看到发送的消息（花、火箭、文字）就可以使用Redis的发布订阅功能

### 监听以后，别人发送消息，才能接受到

    help @pubsub
    
    ====一个窗口连接redis客户端
    redis-cli
    PUBLISH ooxx hello   //发送消息
    
    
    ====另一个窗口连接redis客户端
    redis-cli
    SUBSCRIBE ooxx  //接受消息 
    

### 延伸一下：钉钉、QQ、微信还可以看之前的消息，聊天室的最新数据以及过往数据，往什么地方存呢？

放在mysql中可以保证全量存储；但是多人查的时候，查询分页的成本就比较高

思路扩展一下：

客户端可以接受到  实时性的数据  ，可以查看历史性的数据；历史性数据又分为 3天以内，以及更高的数据，数据肯定是全量存放在数据库的，那么如何解决多人查询的时候，数据库查询的成本呢，就是要用redis做缓存

redis做缓存 ，目的就是解决数据读请求 ，就是为了快；可以存放收藏、点赞、浏览数这些准确性要求不高的数据

1.对于实时性的数据，采用Redis的发布订阅

2.对于历史-3天之内的数据，采用sorted_set(要求有序)，

可以以日期为key，将用户发送的数据作为value存入；

超过3天的数据，日期小的数据下标靠前，可以使用ZREMRANGEBYLEX删除(删除名称按字典由低到高排序的成员)，

3.更老的数据，采用数据库查询

### 架构方案一：

![401](27540D807C81443B92991CC77BDFE567)

一个客户端发送的消息，第一要往发布订阅中放，第二要往sorted_set中放(发送数据的第二天也能看到昨天的数)

一种方案是**单调往发布订阅中放，
线性的插入到sorted-set中；
再单调一次kafka，通过kafka把消息写到数据库**（这么多人的并发，消息很多，写入数据库慢，通过kafka慢慢写入数据）

**这种方案的问题：发送的客户端挂了以后，别人看到发布订阅的数据了，但是还没有写到sorted_set 和数据库中**

### 架构方案二

![402](87635C6C751B48838E3B75E55FB6E05E)

另一种方案：**一个redis的进程在一台服务器只负责发布订阅，
另外一个客户端可以订阅消息；
再用一个redis进程也去订阅通道，将接受到的数据放到sorted_set中；
再来一个微服务service将发布出来的数据订阅走，放到kafka，通过DBservice将kafka中的数据放到数据库中**

# 事务

==使用Redis请记住：Redis的特征是快，如果使用了其他的技术或使用不当，导致Redis变慢，就不建议使用Redis==

**Redis的事务追求的是速度，没有事务回滚的事情**

命令exec ,redis会顺序执行，要么成功要么失败，回滚不是100%的

因为redis是单进程的，多个客户端发送的命令到达队列后，是顺序执行的

如果客户端1和客户端2同时操作redis中的一个内容，开启了事务;队列的命令为：client1:exec、client2:exec、client1:get k1、client2:del k1、client2:multli（开启事务）、client1:multli（开启事务）

从标记开始，一个客户端发来的东西，标记后是放在一个缓存区的，标记后先出现exec命令的先执行，所以客户端2的命令先执行，也就是把k1删除了，客户端1的命令到达后，get k1 就会报错

**MULTI (开启事务)、 EXEC(执行事务) 、 DISCARD （取消事务）和 WATCH（乐观锁实现CAS操作） 是 Redis **事务相关的命令****

    help @transactions
    
    MULTI  //开启事务
    set k1 aaa  //没有执行，只是放到队列queued中
    set k2 aaa
    exec  //执行事务
    
    ================
    客户端1
    MULTI
    get k1
    exec  //后执行，返回nil
    
    
    客户端2
    MULTI
    del k1
    exec  //先提交
    
    ===================WATCH
    客户端1
    set k1 232323
    WATCH k1
    MULTI
    get k1
    keys *
    exec  //后执行，返回nil，整个事务没有被执行
    
    
    客户端2
    MULTI
    keys * 
    set k1 11111
    exec  //先提交
    get k1 //值为11111
    
## 为什么 Redis 不支持回滚（roll back）

如果你有使用关系式数据库的经验，那么“**Redis在事务失败时不进行回滚，而是继续执行余下的命令**”这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：

**Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现）**，或是命令用在了错误类型的键上面：这也就是说，==从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中==。

**因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速**。
==有种观点认为 Redis 处理事务的做法会产生 bug==， 然而需要注意的是， ==在通常情况下， 回滚并不能解决编程错误带来的问题==。 举个例子， **如果你本来想通过 INCR 命令将键的值加上 1 ， 却不小心加上了 2 ， 又或者对错误类型的键执行了 INCR ， 回滚是没有办法处理这些情况的。**

# RedisBloom布隆过滤器


在Redis的英文官网， **顶部导航栏多了"modules"**,多了一些模块，其中有一个RedisBloom

Bloom过滤器是做什么的呢？**在RedisBloom 的github的readme中看到make’编译后生成redisbloom.so文件**

**补：linux的扩展库的后缀是.so   在windows中扩展库的后缀是.dll**

## 安装、启动RedisBloom

通过命令：$ redis-server --loadmodule /path/to/redisbloom.so 将扩展库加载到redis中

    cd    //回到根目录
    cd soft/
    mkdir bf 
    cd  bf
    wget https://github.com/RedisBloom/RedisBloom/archive/master.zip  //下载后是master.zip
    
    yum install unzip  
    unzip master.zip
    
    cd RedisBloom-master/    //自带Makefile文件
    
    make //前提有gcc编译器  yum install gcc;make成功会多一个redisbloom.so扩展库
    
    cp redisbloom.so   /opt/mashibing/redis5  //拷贝扩展库到redis安装目录
    
    cd /opt/mashibing/redis5
    
    service redis_6379 stop
    
    ps -fe | grep redis
    
    redis-server --loadmodule /opt/mashibing/redis/redisbloom.so /etc/redis/6379.conf  
    //启动redis，挂载扩展库;注意扩展库必须是绝对路径，否则无法加载扩展库
    
    ps -fe | grep redis
    
    redis-cli   //连接6379，就有布隆过滤器相应的命令  BF.***
    
    
    ====另一个窗口
    service redis_6380 start
    redis-cli -p 6380  //连接6380 ，没有布隆过滤器的命令
    
    ===========布隆过滤器命令
    BF.ADD ooxx abc
    
    BF.EXISTS ooxx abc  //返回1
    
    BF.EXISTS ooxx dfaf  //返回0
    
    //还有CF开头的命令，其中有CF.DEl
    

## 缓存穿透
    
**为什么要使用布隆过滤器，就是为了防止缓存穿透**

什么是缓存穿透？用户查询的数据，Redis没有，数据库也没有；没有的数据相应的查询，会直接到达数据库，**数据库做很多没用的事情**；例如：黑客攻击，搜索一些网站没有的东西，数据库白白的建立连接，请求，处理搜索；

## 实现原理

![403](1F42004F509741908936F6B0BEBE6ED2)

怎么防止缓存穿透呢？使用布隆过滤器；将数据库中有的数据保存到redis，客户访问的数据有的话，就返回或查询数据库，没有的话就返回没有

但是数据库的数据量很大，怎么实现呢？bitmap

==这段描述有问题==数据库已有的数据，经过k个不同的映射函数，生成的hash，如果值为1，则将bitmap对应的位标记为1

==注意：如果查询的元素，经过k个映射函数到bitmap去匹配,只要有一个为0，就会被过滤器挡住；如果全部为1(碰巧  碰撞了)，就会穿透==


==概率解决问题，不可能百分百阻挡，可以使缓存穿透的概率小于1%==

> 步骤：
> 
> 1，你有啥
> 2，有的向bitmap中标记
> 3，请求的可能被误标记
> 4，但是，一定概率会大量减少放行：穿透
> 5，而且，成本低


## 架构方案

方案一：客户端（实现bloom算法，自己承载bitmap ，也就是胖客户端）  +  redis()

方案二：客户端（实现bloom算法）  +  redis(承载bitmap)

**方案三：客户端()  +  redis(实现bloom算法，承载bitmap)**

**推荐方案三：这样才符合微服务的概念，客户端只做业务，也符合service mesh**

## 作业：了解布隆、count、布谷鸟过滤器(扩展知识)

过滤器有三种，

bloom布隆过滤器

counting bloom

cukcoo布谷鸟过滤器

有兴趣了解一下，面试时可以多讲几句，增加面试分；不需要深入钻研

## 总结

    1，访问redis.io
    2,modules
    3,访问RedisBloom的github  https://github.com/RedisBloom/RedisBloom
    4，linux中wget  *.zip
    5,yum install unzip
    6,unzip *.zip
    7,make
    8,cp bloom.so  /opt/mashibing/redis5/
    9,redis-server --loadmodule  /opt/mashibing/redis5/redisbloom.so
    
    10 ,redis-cli  
    11,bf.add  ooxx  abc
    bf.exists   abc
    bf.exists  sdfsdf
    
    
    12,cf.add   #  布谷鸟过滤器

## 补充知识点

**1.如果布隆过滤器被穿透了，也就是数据库中不存在相应的数据：在redis中增加相应的key,value为error或null的标记**

2.**数据库增加了元素，必须完成元素对Bloom的添加**

**先做一个铺垫；这里面有很多坑，会出现双写。**

# 缓存LRU

## redis作为数据库/缓存的区别：尤其缓存！！！！

redis作为数据库/缓存的区别：

    1.缓存数据"不重要"
    2.缓存不是全量数据
    3.缓存应该随着访问变化
    4.应该存放热数据

如果使用Redis作为缓存，redis里的数据怎么能随着业务变化，只保留热数据？

因为内存大小是有限的，也就是瓶颈

## 内存应该多大呢？内存满了以后怎么淘汰

因为业务逻辑，推导出key的有效期，有效期可能为几秒、分钟、小时；

业务运转：因为内存有限，随着访问的变化，应该淘汰掉冷数据

**那么内存应该多大呢？就是通过配置redis配置文件中的maxmemory来设置**

**场景不同，根据过往经验，最好控制在1G到10G之间;不要太大，后面涉及半持久化存储的时候成本很大，而且数据迁移的成本也很高**

内存满了以后，设置maxmemory-policy 淘汰策略

LFU   碰了多少次

LRU  多久没碰他


### 补Redis的配置文件

    vi /ect/redis/6379.conf

    首先看include，因为redis在一台机器上有多个实例，有些是通用配置，有些是个性化配置，所以
    # include /path/to/local.conf
    # include /path/to/other.conf
    
    load modules
    # loadmodule /path/to/my_module.so  //**注意是绝对路径
    # loadmodule /path/to/other_module.so

    # bind 192.168.1.100 10.0.0.1 //只允许这个ip地址的请求访问
    
    proteced-mode yes //允许外部主机的人访问我，类似mysql数据库的远程访问
    
    ###########GENERAL全局配置##############
    daemonize yes //yes为后台服务模式, no就是前台阻塞
    
    pidfile /var/run/redis_6379.pid  //pid文件的位置
    
    loglevel notice //日志级别
    
    # logfile /var/log/redis_6379.log  //日志文件的位置
    
    databases 16 //redis默认为16个库，从0到15号
    
    
    ###########RDB 先跳过##############
    
    ###########主从复制 先跳过##############
    
    
    ###########SECURITY 安全##############
    
    # requirepass foobared //登录密码
    
    # rename-command CONFIG "" //重命名命令，例如将FLUSHALL FLUSHDB命令重命名
    
    ###########CLIENTS##############
    # maxclients 10000  //最大允许的连接数
    
    ###########MEMORY MANAGEMENT内存管理##############
    
    # maxmemory <bytes> //最大内存，场景不同，根据过往经验，最好控制在1G到10G之间 ;不要太大，后面涉及半持久化存储的时候成本很大，而且数据迁移的成本也很高
    
    # maxmemory-policy noeviction  //内存满以后的淘汰策略：

### LRU淘汰策略 以及选择

    # maxmemory-policy noeviction  //内存满以后的淘汰策略：
        LRU means least Recently Used 最久没有使用，多久没有使用
        LFU means least Frequently Used 最少使用，使用了多少次
        
        可以在中文文档中查看：将redis当做使用LRU算法的缓存来使用
        
        volatile-lru：尝试回收最少使用的键（LRU），但仅限于在过期集合的键,使得新添加的数据有空间存放。
        allkeys-lru：尝试回收最少使用的键（LRU），使得新添加的数据有空间存放。
       
        volatile-random：回收随机的键使得新添加的数据有空间存放，但仅限于在过期集合的键。
        allkeys-random：回收随机的键使得新添加的数据有空间存放
        
        volatile-ttl：回收在过期集合的键，并且优先回收存活时间（TTL）较短的键,使得新添加的数据有空间存放。
        noeviction：返回错误当内存限制达到并且客户端尝试执行会让更多内存被使用的命令（大部分的写入指令，但DEL和几个例外）
        
        volatile-lfu：
        allkeys-lfu：

**LRU means least Recently Used 最久没有使用，多久没有使用**

**LFU means least Frequently Used 最少使用，使用了多少次**

==volatile-ttl:算法成本太高； volatile-random、allkeys-random又太随意==

**一般使用volatile-lru 、allkeys-lru，**

==具体怎么选择：如果大量的缓存设置了过期时间，那么优先选择volatile-lru==

==如果没有怎么设置过期时间，需要靠redis来腾空间，那么就选择allkeys-lru==



## key的有效期

    ==================有效期会随着访问延长？不对！！
    FLUSHALL
    set k1 aaa EX 20 //设置20秒过期
    ttl k1 //查看剩余的有效期
    get k1 //访问的时候，有效期也会减少
    
    ==================发生写，会剔除过期时间
    set k1 aaa  
    ttl k1 //返回-1，永久有效
    EXPIRE k1 50  //设置50秒过期
    set k1 bbb //注意：写以后，又变成永久有效
    
    
    ==================倒计时
    TIME  //查看毫秒值
    EXPIREAT k1  xxx+10秒
    ttl k1  //可以看到倒计时
    
    
    
**易错题：1，有效期会随着访问延长？不对！！**

==易错题：2，发生写，会剔除过期时间==

==3，倒计时，且，redis不能延长==

==4，定时==

5，过期时间，业务逻辑要自己补全

## 过期判定原理：

中文官网：过期（Expires）  ->   **Redis如何淘汰过期的keys**

Redis keys过期有两种方式：被动和主动方式。
    
当一些客户端尝试访问它时，key会被发现并主动的过期。
    
当然，这样是不够的，因为有些过期的keys，永远不会访问他们。 无论如何，这些keys应该过期，所以**定时随机测试设置keys的过期时间**。所有这些过期的keys将会从密钥空间删除。
    
==具体就是Redis每秒10次做的事情：==
    
    1.测试随机的20个keys进行相关过期检测。
    2.删除所有已经过期的keys。
    3.如果有多于25%的keys过期，重复步奏1.
    
这是一个平凡的概率算法，基本上的假设是，我们的样本是这个密钥控件，并且我们不断重复过期检测，**直到过期的keys的百分百低于25%,这意味着，在任何给定的时刻，最多会清除1/4的过期keys。**


过期判定原理：

**1，被动方式：被动访问时判定**

**2，主动方式：周期轮询判定（增量）**

==这么做的目的：稍微牺牲下内存，但是保住了redis性能为王！！！！==

**面试回答时，一定要记得扣题，redis的特征就是快**