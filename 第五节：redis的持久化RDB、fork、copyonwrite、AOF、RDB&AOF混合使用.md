[TOC]

# 缓存常见问题：

击穿

雪崩

穿透

一致性（双写）

==技术是易与人的使用！理论是极其复杂！这样的技术才有生存力==

## 击穿
## 雪崩
## 穿透
## 一致性（双写）

# Redis做缓存、数据库

Redis做缓存：数据可以丢  急速！

Redis做数据库：数据绝对不能丢的  特征：速度+持久性  

内存中数据掉电易失！

**方案：redis+mysql  >  数据库  < 这种做法不太对**

**有人提出这种方案：Redis和mysql都存储同样的数据，这种做法不太对，因为会造成数据一致性的问题**

**如果使用Redis做数据库，不要开启mysql和redis的追平一致性(强一致性)，一般采用异步mysql去写，中间加消息队列**

# Redis单机持久化

![503](4D5A6D6C9D1C4FA9BA3F85BD8C84CBDE)

**提到持久化，有一个通用知识：存储层有1.快照/副本；2.日志**

将Redis内存中数据，每一小时挪一份到硬盘上，这时候在磁盘上有一份副本/快照；再挪出主机，挪到别的主机，便于主机损坏以后的数据恢复

增删改的操作记录日志，根据操作日志，进行数据恢复

**Redis中RDB就是快照/副本， AOF就是日志**


## RDB


### 时点性

![502](6A671990A9D040EA94ECAA46C1AF1D28)

**RDB有一个特性是时点性，**

每隔1小时落一份数据到磁盘，但是数据落到磁盘是需要时间的；

例如，如果8点开始落磁盘，8点10分执行完，那么落到磁盘的数据是8点 还是8点10分的  还是8点到8点10分的记录，具体又是怎么实现的呢？

现在内存中有数据a=3 b=4 ,将这份数据落到磁盘，代码怎么实现呢？

#### 第一种方案：阻塞（容易实现的），redis进程不对外提供服务了
    
#### 第二种方案：1.非阻塞，redis进程继续对外提供服务；2.将数据落地

        例如8点的时候a=3 b=4 ,8点的时候先将a=3落到磁盘、8点10分的时候a=8 b=6了，这时候如果将b=6落到磁盘的话，
        
        如果将落到磁盘的文件恢复到8点时，a=3没问题，但是b=6了；造成了时点混乱！
    
**一个进程既要满足修改，又要满足持久化的，成本很大，并且很难保证时点，落下的数据 时点混乱了**
    
#### 管道

![501](8068D09A30DB4C619B51E4477D3633BC)

讲第三种方案之前，先补一个知识点：管道

    ls -l /etc  //查看etc下的文件太多了，那么怎么一点点查看呢？
    ls -l /etc | more  //使用管道more：点击空格，翻屏，慢慢看
    
    num=0 //定义变量
    echo $num //打印0
    
    ((num++)) 
    echo $num  //打印1
    
    ((num++))  | echo ok  //注意管道前面的输出，没有作为管道后面的输入
    echo $num  //打印1
    
    ================
    echo $$ //打印当前进程的pid
    
    echo $BASHPID  //打印当前进程的pid
    
    echo $$ | more  //打印当前进程的PID，因为$$ 高于 |  ；先看到了$$，就把$$替换成PId，再看到管道，然后开辟子进程
    
    echo $BASHPID |  more  //打印子进程的PID   ；先看到管道，开辟子进程；再看到$BASHPID，也就是子进程的PID
    
管道：

1，衔接，前一个命令的输出作为后一个命令的输入

2，管道会触发创建【子进程】
    
echo $$  |   more

echo $BASHPID |  more

$$ 高于 |  

通过管道引出了父子进程

#### linux的父子进程

    echo $$   //PID=114017
    
    echo $num  //打印1
    
    /bin/bash  //启动一个新的bash进程
    
    echo $$  //PID=114120  子进程id
    
    pstree  //可以看到sshd -sshd--bash-bash-pstree  pstree上面是有两个bash进程的
    
    echo $num  //打印空 ，说明：常规思想，进程是数据隔离的！

    =============

    exit  //退出子进程
    
    pstree //可以看到sshd -sshd--bash-pstree  pstree上面是有一个bash进程的
    
    echo $num  //回到主进程，打印1
    
    export num //num放入环境变量
    
    /bin/bash  //再次进入子进程
    
    pstree  //验证进入了子进程
    
    echo $num //打印了1 ；进阶思想，父进程其实可以让子进程看到数据！
    
    ========================子进程的修改不会破坏父进程
    vi test.sh //写一个脚本
    
        #!/bin/bash     //启动一个子进程bash
        
        echo $$    //打印子进程id
        echo $num   //打印父进程的num
        num=999     //赋值
        echo num:$num   
        
        sleep 20    //睡眠20秒
        
        echo $num
    
    chmod +x test.sh   //修改权限，变绿，说明可以执行了
    
    ./test.sh  &     //添加&，代表脚本后台执行，按下enter执行；按下enter回到父进程
    
    echo $$
    echo $num  //结果还为1
    
    999  //子进程睡眠结束，打印999
    
    ========================父进程的修改也不会破坏子进程
    echo $$  //打印父进程PID
    echo $num
    
    ./text.sh &
    
    echo $num
    num=888   //在20秒内，父进程赋值为888
    
    999   //子进程睡眠结束，打印999
    
    

使用linux的时候：父子进程

父进程的数据，子进程可不可以看得到？

**常规思想，进程是数据隔离的！**

**进阶思想，父进程其实可以让子进程看到数据！**

**linux中export的环境变量，子进程的修改不会破坏父进程**

**父进程的修改也不会破坏子进程**


#### 创建子进程：系统调用fork

**创建子进程的速度应该是什么程度？**

如果父进程是redis，内存数据比如10G，有两个影响因素：

1，速度 ;2，内存空间够不够

**这两个问题的解决，linux有系统调用fork** : 1，速度：快 ;2，空间：小


计算机有一个内存，是物理内存，可以想象成一个线性的地址空间(字节数组)

程序默认认为内存都是自己用的，所以会有一个虚拟地址

假设有一个redis，有虚拟地址空间，虚拟地址到物理地址的映射（ mmu映射 redis中有一个变量a,虚拟地址为3；指向的物理内存的地址为8，值为x）

**redis要创建一个子进程，也有虚拟地址；数据拷贝有两个方案：1.可以将数据拷贝过去；2.更快的方案：fork（fork不是真正拷贝数据，只是将子进程的虚拟地址到物理地址做一个绑定；绑定的成本不高，只是指针的复制）**

之前讲到的，==其中一方数据的修改另一方看不到；这个涉及到一个知识点：copy on write==

#### copy on write：内核机制

copy on write 写时复制：内核机制

**意思就是：创建子进程并不发生复制**

好处：创建进程变快了；同时根据经验，不可能父子进程把所有数据都改一遍(触发copy on write的概率不高)

**父进程修改数据的时候，是先在物理内存创建好要修改的值w(物理地址为9)，然后把父进程、子进程中的指针指向新的物理地址9**

也就是说玩的是指针


#### 总结

上面的知识点，可以在redis帮助文档中看到

    man 2 fork  //fork-create a child process
        //fork() is implemented using copy-on-write pages
    
#### 方案三：redis实现:fork+copy-on-write    

redis的实现：当你的程序有了这些数据以后，会创建出一个子进程；也就是在8点的时候**通过fork()创建出一个子进程，速度很快，且数据没有真的拷贝出来**(虽然redis有10G)

**这时候父进程(你的程序)对数据做了修改，会触发一个copy-on-write写时复制，那么修改的值先进入物理内存，然后父进程的指针指向新的物理地址; 此时子进程的值还指向修改之前的物理地址**

数据写入磁盘是由子进程来负责的，**子进程看不到父进程修改(增删改)的内容,不管子进程落到磁盘的时间有多长，它落的数据永久都是8点拷贝出来的那份指针**；也就保证了==时点正确！==


==父进程负责对数据进行增删改 ；子进程负责数据(不会对数据进行增删改)写到磁盘==

==也就是redis的实现就是系统调用fork + copy-on-write写时复制==

**系统调用fork 可以快速创建子进程，并且数据还没有涨出内存(只是复制了指针)； 但是没有达到数据隔离**

**为了达到数据隔离，采用了copyonwrite写时复制**

## 实现方式

### 1.人为触发的实现方式：1.命令：save 、bgsave

**命令save：明确：适用场景：关机维护（save执行后，开始阻塞，并且不对外提供服务了）**

**命令bgsave:后台执行，触发fork创建子进程**

### 2.配置文件触发：配置文件中给出bgsave的规则： 使用save这个标识

注意虽然是save标识，实际执行的是的bgsave

    cd /etc/redis
    vi 6379.conf
        ##############SNAPSHOTTING快照################
        # save <seconds> <changes>  //时间  操作数
        
        //默认配置
        save 900 1
        save 300 10
        save 60 10000  //条件：60秒或1万个操作数，两个条件都满足，就会触发RDB
    
        # save ""  //关闭RDB
        
        rdbchecksum yes //rdb后面会一个校验位
        
        dbfilename dump.rdb  //RDB文件的名称
        
        dir /var/lib/redis/6379 //RDB文件的位置

## 弊端

1.**不支持拉链：因为只有一个dump.rdb**

==需要运维人员去做定时策略，每天将最后一个RDB拷贝出去，并加上时间保存历史版本==


2.**丢失数据相对多一些：因为时点与时点之间窗口数据容易丢失**

8点得到一个rdb，9点刚要落一个rdb，挂机了

跟HDFS 的Fimage是一样的

**mysql数据库做异地存储，也会出现丢失数据问题**

**全量数据的持久化备份，会出现丢失数据的问题；**

3.**小优点：RDB类似java中的序列化，恢复的速度相对快**

==RPC中一种方式通过json传输，类型转换；另一种是序列化，拿到数据以后直接反序列化，取到数据==


# AOF


**基于RDB的弊端，有了AOF(Append-only file)只追加操作的文件**

AOF的简单介绍：

**AOF:redis的写操作记录(增删改)到文件中**

==好处就是：丢失数据少==

==redis中，RDB和AOF可以同时开启；需要注意：如果开启了AOF，只会用AOF恢复；==

**Redis4.0以后，AOF中包含RDB全量，增加记录新的写操作**


---

## 讲一个故事：redis运行了10年，并且开启了AOF；10年头，redis挂了，那么有两个问题：1，AOF多大？2，恢复要多久？3.恢复的时候会不会溢出？

1，AOF多大：很大，10T

2，恢复要多久：恢复用5年（记录了10年，恢复需要用5年）

3.恢复的时候会不会溢出？不会溢出

## 弊端：体量无限变大  >  导致 恢复慢

## 设计一个方案让日志，AOF足够小

如果能保住日志，还是可以用的；结果：设计一个方案让日志，AOF足够小

### hdfs的方案：fsimage(类似RDB)+edits.log （让日志只记录增量，有合并的过程，目的就是让日志足够小）

就是日志里记录的东西写到一个镜像文件里，日志就会被清空，镜像就会往前增加时间(从8点变成了9点)


### redis的方案

==重写：减少AOF的体积，把没用的指令剔除掉，方便未来加载变快，不需要执行没有意义的指令==

#### 4.0以前，重写：删除抵消的命令，合并重复的命令；最终也是一个纯指令的日志文件

**重写就是 删除抵消的命令，合并重复的命令**，最终也是一个纯指令的日志文件

抵消的命令：新增一个key、删除一个key

重复的命令：很多的key+1


#### 4.0以后，重写：将老的数据RDB到aof文件中，将增量的以指令的方式Append到AOF；AOF是一个混合体，利用了RDB的快，利用了日志的全量

将老的数据RDB到aof文件中，将增量的以指令的方式Append到AOF

**结论：AOF是一个混合体，利用了RDB的快，利用了日志的全量**

## 写IO的级别

回到原点：redis是内存数据库，写操作会触发IO，会变慢，有三个级别可以调：NO、AWAYS、每秒

    配置文件：
    #############APPEND ONLY MODE##############
    # By default Redis asynchronously dumps the dataset on disk;redis默认异步dump数据到磁盘
    
    appendonly yes //默认是关闭的，需要手动打开

    appendfilename "appendonly.aof"  //文件名称
    
    auto-aof-rewrite-percentage 100   //
    auto-aof-rewrite-min-size 64mb
    
    appendfsync always   
    appendfsync everysec    //默认为每秒
    appendfsync no 
    
    redis程序，是需要调内核的，内核中文件为文件描述符，内核会给文件描述符开辟buffer；buffer满了，内核刷到磁盘，再关闭流
    
    写级别NO：，redis不调flush，内核中的buffer什么时候满了，什么时候刷到磁盘；这样会造成一个问题：可能丢掉一个buffer长度的数据
    
    写级别always：redis向内核中buffer写一次，就调用一次flush，将数据写到磁盘；数据最可靠的，如果断电了，最多丢一条
    
    写级别everysec:每一秒调一次flush；最大丢失数据：接近一个buffer（因为buffer如果满了，就会自己刷到磁盘）；是一个中间方案，丢失数据量介于NO和always
    
    no-append-sync-no-rewrite no //no：如果redis抛出一个子进程，子进程可能在做bgsave，bgwrite,不管写IO是什么级别，父进程都不会调flush将数据刷新到磁盘；它认为自己在对磁盘进行疯狂的写，子进程不能竞争IO写操作；
    //这样容易丢数据，是否开启在于数据敏感性
    
    aof-use-rdb-preamble yes //redis4.0以后才有，重写实现配置；
    //查看描述，开启以后：将老的数据RDB到aof文件中，将增量的以指令的方式Append到AOF;如果AOF文件中以“REDIS”开头，说明是混合文件，否则则是老的RDB文件
    //4.0以后才有该功能，开启后，利用了RDB和AOF的优势，

## 自动触发AOF

    配置文件：自动重写触发规则
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb  //aof文件到达64M，触发重写；
    //redis会记录最后一次重写后的体积大小，
    
    
    
# 验证RDB和AOF

## 实验1：把redis调整为手工运行，并且日志打印在屏幕

    ps -fe | grep redis  //查看redis进程是否正常运行
    
    
    vi /ect/redis/6379.conf  
        daemonize  no  //不是后台服务，变成前台阻塞执行
        
        # logfile /var/log/redis_6379.log //关闭日志文件，将日志打印到屏幕上
        
        //RDB的设置
        save 900 1
        save 300 10
        save 60 10000
        
        appendonly yes //开启追加
        
        aof-use-rdb-preamble no //关闭RDB和AOF的混合模式
        
    cd /etc/lib/redis/6379
    rm -fr ./*   //清除redis的数据
        
    redis-server  /ect/redis/6379.conf    
    
    cd /etc/lib/redis/6379
    ll  //查看到有appendonly.aof文件（并且为空），RDB文件没有


    ============再开一个redis客户端连接6379：
    redis-cli
    set k1 hello
    
    ==============再回来查看数据目录，没有rdb文件，因为没有触发RDB的条目
    
    vi  appendonly.aof  //查看aof文件，AOF的指令协议，
        *2    //*代表有几个元素组成：select 0  ； set k1 hellp
        $6    //$代表元素由几个字节组成
        SELECT
        $1
        0    //选择了0号库
        *3
        $3
        set 
        $2
        k1
        $5
        hello
    
    
    =============第二个redis客户端
    
    bgsave  //手工触发，生成RDB文件 ；不使用save，因为save执行后，开始阻塞，并且不对外提供服务了
    
    //数据目录中多了dump.rdb文件
    //redis的日志中出现了：Background saving started by pid xxxxx (子进程PID)
    
    
    vi dump.rdb //以REDIS开头，说明是混合体，内容是序列化后的二进制，是看不懂的
    redis-check-rdb dump.rdb  //检查rdb文件，如果出错会告诉你哪里出错了
    
    set k1 a 
    set k1 b
    set k1 c   //AOF文件会增长
    
    BGREWRITEAOF   //重写AOF，注意：这是老的版本
    vi appendonly.aof  //验证了重写的作用
    
==重写：减少AOF的体积，把没用的指令剔除掉，方便未来加载变快，不需要执行没有意义的指令==
    
## 实验2：

    关闭redis服务
    rm -fr /etc/lib/redis/6379/*   清空redis数据目录
    
    vi /etc/redis/6379.conf
        daemonize  no  //不是后台服务，变成前台阻塞执行
        
        # logfile /var/log/redis_6379.log //关闭日志文件，将日志打印到屏幕上
        
        //RDB的设置
        save 900 1
        save 300 10
        save 60 10000
        
        appendonly yes //开启追加
        
        aof-use-rdb-preamble yes //开启RDB和AOF的混合模式
    
    redis-server  /ect/redis/6379.conf
    //  /etc/lib/redis/6379中appendonly.aof文件（并且为空）
    
    
    redis-cli
    set k1 a 
    set k1 b
    set k1 c
    set k1 d
    //这是appendonly.aof文件还没有重写，没有变成混合体
    
    BGREWRITEAOF  //重写后，appendonly.aof文件

appendonly.aof文件中多了一个REDIS，说明==多了rdb的存储==；==这样就不用浪费CPU去做前面已有指令的删除抵消、合并重复的指令；减少了CPU指令集的过程==

    set k1 w
    set k1 m
    //appendonly.aof文件中追加了这两条指令的记录
    
这个就验证了：==AOF混合体=全量数据+增量日志==，这种方式恢复速度快，因为数据不是全量的

    bgsave  //生成RDB
    BGREWRITEAOF  

可以看到RDB、AOF文件的时点一致；查看两个文件内容，发现内容一样
    
# 问题

问题1：如果发生误操作，如何找到历史记录

==注意如果发生误操作，能不能找到历史记录，关键就在于有没有做过BGREWRITEAOF重写aof==

    =====redis4.0以后
    set k1 1
    set k1 2
    set k1 3
    FLUSHALL 
    //appendonly.aof文件中记录了操作,包括误操作
    
    BGREWRITEAOF
    //aof文件跟rdb文件一致了(之前的记录保存到rdb中了)，aof中没有了刚才的操作记录，也就无法根据aof记录进行恢复数据了
    
    ==============redis4.0之前
    vi /etc/redis/6379.conf  //aof-use-rdb-preamble no,关闭RDB和AOF的混合模式
    rm -fr /etc/lib/redis/6379/*
    flushall  //通过redis客户端清空数据
    redis-server /etc/redis/6379.conf
    
    set k1 1 
    set k1 2
    set k1 3  //这时候保留了三条记录
    
    BGREWRITEAOF //重写以后只有set k1 3的记录了，删除撤销的命令；没有了set k1 1 和 set k1 2的指令

    
问题2：如果发生误操作，如何恢复数据
    
    set k1 1
    set k1 2
    set k1 3
    FLUSHALL
    
    vi appendonly.aof //编辑aof文件，删除FLUSHALL的指令
    
    重启redis以后，就会误操作之前的记录
    
    


    
    
    
    
    
    
    
    

    
    
    
    
    
    