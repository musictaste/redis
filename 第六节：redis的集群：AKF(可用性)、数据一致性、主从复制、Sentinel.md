[TOC]

# Redis集群方式

## 单机、单节点、单实例可能存在的问题有哪些？

1.单点故障

2.容量有限

3.压力：
    
    Socket IO压力，CPU密集型操作的压力（面试时，考查你是否有项目经验）

## 面试官问：单点故障怎么解决？工作2/3年，刚开始是单机单点的，后来怎么处理的，解决了哪些问题？ -》 AKF

## AKF

![601](73EF65A700A74365A93276C85C987268)

**面试再问到单点故障如何解决问题，可以往AKF上说**

redis是一台的，基于本地的RDB、AOF(磁盘)会有数据可靠性

但是物理机还有可能挂，那么单点故障应该怎么解决呢？

**延X轴，做数据库的副本**

客户端可以只访问一个独立的redis，增删改都发生在这台redis上；这台redis挂了以后，可以连接后面的redis，所以备用性就有了

**既然已经有了多台redis，那么就可以把增删改放在一台redis，查询放在备用的redis上，也就是读写分离了**


---


那么这个方案可以解决 容量有限、压力的问题吗？

首要要明白：AKF

==基于X轴：全量、镜像==
    
    那么容量有限的问题是解决不了的；
    而压力的问题也只能解决读数据的压力(可以访问多台redis读数据)，写数据的压力解决不了

==基于Y：业务，功能==

    ==多台redis分别负责==：订单、用户信息、页面元素；
    AKF是微服务4个拆分原则的第一项；
    AKF不仅只限定于微服务，数据库，tomcat、redis很多东西都可以应用
    
    基于业务的时候，会在基于X轴做备用

==基于Z：优先级，逻辑再拆分==

    如果用户的数据量变大，那么增加redis，但是一定要注意：要加一定的规则，让出现在不同的数据库里
    

==通过AKF的X/Y/Z就是在综合解决redis单机、单节点、单实例存在的三个问题==

**AKF其中的某一个点，不是承载公司所有的数据了，她的数据量已经足够小，可以发挥单机的性能，也没有了容量的限制，并且还有备用的节点，所以也不会单点故； 另外呢数据量足够小，是公司中的一小片数据，所以访问这片数据的压力也不会太大障**

## AKF带来的问题

### 问题1：数据一致性

![602](68FF355DFB1648C4A30564648E12CEFE)

**通过AKF，实现一变多**

一变多带来的第一个问题：**数据一致性！！！**！

如何保障数据一致性：

第一种方式：**强一致性！:所有节点阻塞直到数据全部一致(同步方式阻塞方式)**

==同步阻塞==

强一致性是企业一直追求的，但是成本极高，或者根本不可能达到

**强一致性带来的小问题：客户端给主的写成功了，主往备机写的过程中，备机进程异常退出了，执行慢了、或网有问题（延迟了，丢包了）**

所以**强一致性容易破坏可用性！**

这时反问自己：==为什么要一变多==，是为了解决单点问题、容量有限、压力，也==就是解决可用性==

那么为了解决可用性，应该怎么解决了呢？**强一致性降级**

方案二：**弱一致性：通过异步方式，容忍数据丢失一部分**


方案三：**最终一致性**

通过kafka这类的东西(不一定是Kafka，也可以是AFS，自己实现的)；**是可靠的，集群，响应速度够快**

**还是客户端访问主redis，主redis数据保存成功以后，直接返回给客户端；保证了响应速度快**

**通过kafka，将数据从主redis发送给备用机，即使中间出现故障，因为kafka可靠，最终可以达到数据一致性**

从主redis到kafka是同步阻塞的

注意最终一致性存在小问题：==客户端访问redis(黑盒化访问)，有可能取到不一致的数据==(因为是最终一致性)

==HDFS采用这种方案==
    
    
### 知识铺垫：主从、过半

主从与主备的区别：

主备：有三台redis，一台为主，那么客户端只能访问主，不能访问备

主从：有三台redis，一台为主，客户端既然访问主，也能访问从

无论是主从还是主备，一定有一个**主**，可以**读**写，主自己又是一个单点

所以需要**对主做HA 高可用**，不对从做HA

想要通过程序或技术来实现自动故障转移(主挂了，把从设置为主)，那么就要行一个问题：人是怎么监控的？

**如果由一个技术、程序实现自动故障转移，那么这个程序也会存在单点故障的问题**

**只要是一个程序就会有单点故障的问题**

那么就需要**一变多的集群**

说到这里，好像又回到了原点单点问题，变成了一个死循环；其实呢**有一些地方是不一样的**

那么不一样的地方是什么呢？讲这个之前，先铺垫一个知识

如果有三个程序监控一个redis，就好比反转过来，数据和状态；也就是一台redis是好是坏，是这三台监控一起决策呢还是部分决策？

结合上面的数据一致性，如果**三台监控都给出结果，那么就是强一致性**

强一致性会导致不可用，也很难实现；那么就降级，弱一致性

**一部分给出结果，另一部分不算数(很关键)**，那么一部分是几个呢？1？2？

推导：

1.如果1个监控给出结果，统计不准确，**不够势力范围**

产生一个问题：网络分区(三台监控，通过网络获取，不同的网络拿到不同的结果)，也就是**脑裂！**

脑裂不一定是坏事，有一个概念：**分区容忍性**

分区容忍性，举例**Springcloud中服务注册发现**是多台，购物车服务有50台，服务注册发现时，有的看到50台可用，有的看到30台可用，实际呢有一台通信就可以了，**这种情况对于数据的一致性要求没有那么高，不一定要过半，有一个能用就可以**


为什么购物车服务会有50台？因为满足高并发，负载均衡；一台tomcat连接1000个连接，50台就可以连接5万个连接，这就是**CAP**

2 在3个节点成功解决脑裂问题

    2个形成势力范围，另1个不算数，只要就能得到明确的结果：是好还是坏

3 在4个节点成功解决脑裂问题

    如果2个形成一个势力范围，另外2个也形成一个势力范围，还是存在脑裂的问题
    
    变成3就解决脑裂的问题了

3 在5个节点成功解决脑裂问题


==(n/2)+1   过半！使用奇数台！==

### 作业：了解CAP原则

百度：CAP原则

==CAP原则又称CAP定理，指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。==

**上面的知识点提到了 强一致性 破坏了 可用性**

**大数据的HBase中也有CAP原则，HBase强调 一致性(强一致性)、分区容错性(允许一些分区出现问题)；可用性（一个分区、分片出现问题，那么这个分区整体下线，不允许访问；活着的正常访问，出现问题的不允许访问）**

**Redis也是，如果一个分片出现问题，整个集群进入维护模式**

## 主从复制

redis中文官网--“复制”

当你知识体系足够宽，就具备了自学的能力

==Redis使用默认的异步复制，其特点是低延迟和高性能==，**也就是采用数据一致性的弱一致性**

这样做的目的：保证快

==**保证快，减少技术的整合，没有引入中间件，自己使用异步的方式完成了通信**==

### 演示
    cd    //回家，/root
    mkdir test
    cd test
    
    # 为了模拟，准备三台redis
    cd   //回家
    cd /soft/redis-5.0.5/utils
    ./install_server.sh   //6381
    service redis_6381 stop
    
    cp /etc/redis/* /root/test   //拷贝配置文件到test
    
#### redis前台阻塞运行，没有AOF，只有RDB
    vi redis_6379.conf
        daemonize no //前台阻塞运行
        # logfile /var/log/redis_6379.log  //注释掉日志文件
        appendonly no  //关闭AOF
        
    vi redis_6380.conf
        daemonize no //前台阻塞运行
        # logfile /var/log/redis_6380.log  //注释掉日志文件
        appendonly no  //关闭AOF    
    
    vi redis_6381.conf
        daemonize no //前台阻塞运行
        # logfile /var/log/redis_6381.log  //注释掉日志文件
        appendonly no  //关闭AOF    
        
        
    cd /var/lib/redis   //持久化目录
    rm -fr  ./6379/*   //清空数据
    rm -fr  ./6380/*
    rm -fr  ./6381/*
    
    redis-server ~/test/6379.conf
    
    
    #再开一个标签
    redis-cli -p 6379
    keys * 
    
    ====================
    
    #再开一个标签
    cd test
    redis-server ./6380.conf
    
    #再开一个标签
    redis-cli -p 6380
    keys * 
    
    
    ====================
    
    #再开一个标签
    cd test
    redis-server ./6381.conf
    
    #再开一个标签
    redis-cli -p 6381
    keys * 
    
#### 启动三台redis实例，期待6379为主机，6380、6381为从机
    help SLAVEOF  //4.0之前的命令
    help REPLICAOF //4.0之后的命令
    
    
    # 6380的实例
    REPLICAOF 127.0.0.1 6379  //设置从机6380，追溯6379
        //可以在6379日志中看到：
            Replica 127.0.0.1:6380 asks for synchronization
            Starting BGSAVE for SYNC with target:disk
            Background saving started by pid xxxxx
            Db saved on disk
            RDB:6 MB of memory used by copy-on-write
            Background saving terminated with success
            Synchronization with replica 127.0.0.0.1:6380 succeeded
        //在6380日志中看到
            MASTER <-> REPLICA sync: Flushing old data //将从机劳的数据清除
            MASTER <-> REPLICA sync: loading DB in memory //将主机的RDB加载

并且在6380的dump.rdb文件中可以看到repl-id(追随的主机id)
    
    # 数据操作验证
    set k1 aaa  //6379设置k1
    get k1  //6380获取k1
    
==默认从机是不允许写数据的，是READONLY==的，可以调
    
    
    # 6381
    REPLICAOF 127.0.0.1 6379  //同样可以通过日志来观察

#### 从机挂掉，重启使用RDB，从机的数据是增量同步

    # 模拟：从机挂掉？那么从机的数据是全量从主机拉一份呢？还是在现有的基础上，增量添加
        # CTRL +C 6381退出
        
        set k1 afaf //6379添加一些数据
        
        redis-server ./6381.conf --replicaof 127.0.0.1 6379 //启动的时候，告诉的追溯主机
    
启动时带参数：--replicaof，在6379中没有看到RDB的操作，并且6380可以看到挂掉期间的数据，这就是==增量同步过程==
    
#### 模拟2：6381再次挂掉，重新启动，并开启AOF，是全量同步

    redis-server ./6381.conf --replicaof 127.0.0.1 6379 --appendonly yes  //appendonly:开启AOF文件

--replicaof  --appendonly，看到了RDB的操作，并且如果6381再次挂掉，再以这样的命令启动，还是可以看到RDB落到从机的过程（主机没有增加任何数据）
    
redis如果开启了AOF，是不碰RDB的；虽然现在AOF是混合模式，上半段是RDB，下半段是记录增量的数据；

==AOF和RDB还是有本质的区别的：RDB可以记录我曾经追随的主机(IP号)，但是AOF是不会记录这个数据的（不知道是不是版本的bug）==

==如果开启appendonly=yes,整个流程是：主把RDB同步给从机，从机加载到内存中；从机再重写到一个AOF文件中;并且dump.rdb中没有repl-id（追随的主机id）==

**不知道是bug，还是设计就是如此**

==只需要记住：开启AOF是全量同步；RDB是增量同步==

#### 主机挂掉

记住：主机可以知道它有几个从机
    
主机挂掉，从机可以查询数据，但是不能插入数据
    
##### 人为设置6380为主，6381为从
    
    REPLICAOF on one //6380不在追随任何机器，即变为主机
        //日志中看到MASTER MODE enabled
        
    REPLICAOF 127.0.0.1 6380  //设置6381追随6380
        //6380日志有落RDB的操作
        
    keys * //两台数据都在
    
    
### 主从复制-配置

![603](A400BD6C0239453B8A6045A47C9CBC81)

    vi /test/6381.conf 
    
    //找到replication  ,Master: receive + writes  ;   Replica:exact + copy  但是可以修改
    
    ############REPLICATION############
    
    # replicaof <masterip> <masterport> //设置追随的主机
    
    # masterauth <master-password> //主机的密码
    
    replica-serve-stale-data yes  //是否同步完数据才允许查询; 
    //当从机启动，并且已经追随主机；如果主机数据量大，同步需要时间，这个时候是否允许查询老的数据,no=不允许，yes=允许查询
    
    replica-read-only yes //从机是否只读
    
    repl-diskless-sync no 
    //no:主机通过磁盘IO(几百兆)将RDB落到磁盘，再通过网络IO传输到从机
    //yes:主机直接通过网络（光纤几G）将RDB传输给从机
    
    repl-backlog-size 1mb //增量复制
    //主机同步数据到从机：主机将数据落到RDB，将RDB传输给从机，从机再将RDB加载到内存
    //主机中还有一个队列
    //这是如果从机挂掉了，又快速恢复了，为了避免全量同步；会从RDB拿到一个offset偏移量，例如8，去主机查询，主机队列的偏移量为20，则将8到20的数据同步过来
    //需要注意：这个值的大小需要根据业务来决定；如果你的业务增删改的操作不多，从机短时间故障，操作的数据没有大过你设置的队列大小，那么没有问题；
    //如果你业务的增删改操作特别多，那么虽然从机短时间故障，但是队列的数据已经被刷新，那么你无法通过偏移量去做增量同步，只能做全量同步
    
    # min-replicas-to-write 3 //最小写几个才能写成功
    # min-replicas-max-lag 10
    //默认是注掉的，因为开启会将异步同步的弱一致性 变成强一致性
    //如果关心数据的数值和一致性的话，可以设置
    
## 主从复制高可用-Sentinel哨兵

![606](EDEBC2F455184E0D823A4A0B41F5CA18)

因为主从复制，需要人工维护主的故障问题，所以需要主从复制高可用

官方文档-高可用 学习    

**Redis 的 Sentinel 系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：监控、提醒、自动故障迁移**

上面提到的主从复制-对主做HA，自动故障转移时提到的监控就是哨兵

### 启动 Sentinel

    对于 redis-sentinel 程序， 你可以用以下命令来启动 Sentinel 系统：
    对于 redis-server 程序， 你可以用以下命令来启动一个运行在 Sentinel 模式下的 Redis 服务器：
        redis-server /path/to/sentinel.conf --sentinel
    两种方法都可以启动一个 Sentinel 实例。

    启动 Sentinel 实例必须指定相应的配置文件， 系统会使用配置文件来保存 Sentinel 的当前状态，并在Sentinel重启时通过载入配置文件来进行状态还原。

    如果启动 Sentinel时没有指定相应的配置文件，或者指定的配置文件不可写（not writable）， 那么 Sentinel 会拒绝启动。
    
### 配置 Sentinel

    Redis 源码中包含了一个名为sentinel.conf的文件，这个文件是一个带有详细注释的 Sentinel 配置文件示例。

    运行一个 Sentinel 所需的最少配置如下所示：
        sentinel monitor mymaster 127.0.0.1 6379 2
        sentinel down-after-milliseconds mymaster 60000
        sentinel failover-timeout mymaster 180000
        sentinel parallel-syncs mymaster 1
        
        sentinel monitor resque 192.168.1.3 6380 4
        sentinel down-after-milliseconds resque 10000
        sentinel failover-timeout resque 180000
        sentinel parallel-syncs resque 5
        
    第一行配置指示Sentinel去监视一个名为mymaster的主服务器，这个主服务器的 IP 地址为127.0.0.1，端口号为6379，
    而将这个主服务器判断为失效至少需要 2 个 Sentinel同意（只要同意Sentinel的数量不达标，自动故障迁移就不会执行）。

    不过要注意， 无论你设置要多少个Sentinel同意才能判断一个服务器失效，一个 Sentinel 都需要获得系统中多数（majority） Sentinel 的支持， 才能发起一次自动故障迁移， 并预留一个给定的配置纪元 （configuration Epoch ，一个配置纪元就是一个新主服务器配置的版本号）。

    换句话说， 在只有少数（minority） Sentinel 进程正常运作的情况下， Sentinel 是不能执行自动故障迁移的。
    
### 演示：

    cd /test  //中已经有6379、/680、6381的配置文件
    vi 26379.conf
        port 26379
        sentinel monitor mymaster 127.0.0.1 6379 2
        
    cp 26379.conf 26380.conf
    cp 26379.conf 26381.conf
    
    # 修改端口号
    vi 26380.conf
        port 26380
    vi 26381.conf
        port 26381
        
    # 启动redis，设置主从
    redis-server ./6379.conf
    redis-server ./6380.conf --replicaof 127.0.0.1 6379
    redis-server ./6381.conf --replicaof 127.0.0.1 6379
    
    # 启动哨兵
        //源码目录/soft/redis-5.0.5/src中有命令：redis-cli   redis-server   redis-sentinel 
        //但是在安装目录/opt/mashibing/redis5/bin  是没有redis-sentinel命令的，需要做一个软接
        //redis-sentinel可以是一个独立的程序，也可以在redis-server中包含redis-sentinel
    
    redis-server ./26379.conf -sentinel
        //启动哨兵，发现日志中监控了+ monitor master，下面还显示监控从机 +slave slave
        
        //那么哨兵是怎么知道两台从机的呢？主机中肯定会记录从机的信息，具体实现又是什么呢？redis的发布订阅
        
    redis-server ./26380.conf -sentinel
        //不仅知道主机、从机、还知道+sentinel 上面的哨兵
    
    redis-server ./26381.conf -sentinel
         //知道主机、两台从机、还知道+sentinel 两台哨兵
         
    # 主机挂掉,查看哨兵自动故障转移
        //刚开始发现哨兵没有变化，是因为基于冯诺依曼、网络IO会有延迟
        //几秒以后，哨兵有变化，其中一个哨兵日志多，另外两个哨兵日志少
        //日志多的哨兵，知道主机挂了，开启新的纪元、投票选出主机6381
        //并且哨兵中的配置文件也修改了
![604](BDD044FEC8FA479FA6699FA04D2A4D3A)

![605](795F361E0FA44FB9AE1FBB83E56C62AB)

### 实现

==哨兵监控主机的时候，因为主机记录了从机的信息，所以知道了从机的信息==

但是==哨兵是怎么知道其他的哨兵呢？是通过redis的发布订阅==

==第一哨兵开启了主机的发布订阅功能，从而可以通过监听消息知道其他哨兵==

PSUBSCRIBE *  //订阅符合正则的通道的消息  PSUBSCRIBE pattern 
