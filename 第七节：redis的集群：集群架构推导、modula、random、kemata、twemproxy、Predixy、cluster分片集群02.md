[TOC]

# 预习作业：Redis API
    
https://sping.io

projects - spring data - spring data Redis

学习方法：通过翻译先快速看3/4遍，有个大概；再通过英文再看一遍

开发人员要学会看官网和Github的readMe

# 回顾上节课

上节课讲了AKF，有x/y/z轴；主从复制，以及HA高可用的哨兵Sentinel

**主从复制是X轴方向的， 有两种使用：主提供读写，从只做数据备份；第二种：主提供读写，从机提供读，实现读写分离**

**单节点的三个问题(单机故障、容量限制、压力)：推导出主从复制，还没有解决容量限制的问题**

# 容量限制问题的解决

![701](D17CEB2CFE944188B8E11EBA1596DED5)

redis管理的内存不建议太大，会影响数据序列化、持久化，为了redis管理的内存不大，那么应该怎么设计呢？

## 方案一：业务拆分  -> 有些数据无法拆分

数据可以分类，交集不多
    
**逻辑：业务拆分**

不同的数据往不同的redis上放，redis在y轴上有多台；需要client做逻辑判断，根据业务做数据拆分，例如商品、订单


## 方案二：hash+取模(modula) ->影响分布式下的扩展性

数据没办法划分拆解，并且数据量就是很大，例如订单数据；
    
**算法：hash + 取模 ->moudula**

取模的数 取决于你后面的redis的数量

数据 进行 sharding分片环节

**弊端：弊端：取模的数必须固定，%3、%4、%10，取模会影响分布式下的扩展性**
    
## 方案三：random-->消息队列

逻辑：random

**弊端：client存进去的数据，自己也不知道存哪台redis里了**

==应用场景：消息队列，客户端请求往redis中随机放数据ooxx，另一台客户端从redis中取数据==
    
**数据ooxx类似于kafka中的topic，redis类似于kafka中的partition**

## 方案四：kemata一致性哈希

![702](0B7C04CAF96144CCAF8D8D81DAEF0C88)

逻辑：kemata一致性哈希
        
一致性哈希 区别于上面的 hash+取模，
    
    因为1.没有取模运算；
    2.对data(写入的数据key)、node(机器的唯一标识)进行hash运算
    
哈希算法得到一个和原来值 等宽 的其他值，跟原有的值做映射

哈希算法你 是一种映射算法，常见的映射算法还有：crc 16位、crc 32位、fnv 、md5
    
一致性哈希：规划一个环形，叫哈希环

### 实际的执行过程是：

    redis节点就是node，node可以把ip地址或机器码等内容作为唯一的标示；这个标示通过哈希算法在 哈希环找到一个点
    
    写入的数据data，也就是redis<k,v>中的key，也通过哈希算法在哈希环上找到一个点
    
    注意环上的点是虚拟的，但是node节点在哈希环上的节点是物理的
    
    物理的点放在treemap中，也就是node得到的点在哈希环上是物理的
    
    data通过哈希算法得到的环上的点datapoint，通过treemap去找大过datapoint点的物理点有哪些，再找到离它最近的物理点，通过这个物理点就可以找到对应的redis节点，就可以把数据放入redis中了
    
    插入的数据恰巧都放在了一个node上
    
### 加一个新的节点

现在在分布式情况下要做扩展，要加入一个新的节点，还是根据node的值早哈希环上找到一个点，但是引发了新的问题：插入的这个节点之后的那个节点的数据，查询时会找到新插入的节点上，导致查询不到；查询不到就会造成缓存击穿

==优点：你加节点，的确可以分担其他节点的压力，不会造成全局洗牌(hash+取模方案会造成全局洗牌)==

==缺点：新增节点造成一小部分数据不能命中==

==1，问题，缓存击穿，访问压到mysql==

==2，解决方案：取离我最近的2个物理节点==

**所以更倾向于 作为缓存，而不是数据库用！！！！**
    
---

    
解决方案：**有的同学可能会想到可不可以做数据迁移，将后面那个节点的数据，迁移到新插入的节点中，不行！！！**

原因：需要将后面那个节点中的key拿出来，重新进行哈希计算，而1.重新计算所花费的时间是不固定的；2.如果在计算的过程中，数据发生了变化(修改，删除)

**这样的解决方案会导致时间成本不可控或时间成本指数级增长**

### 虚拟节点--解决数据倾斜问题

如果只有两个node节点的话，如果插入的数据恰巧都集中一个节点上，这样就会造成数据倾斜
    
解决方案：使用虚拟节点
    
具体过程：一个node后面加1到10中一个数，重新进行哈希计算，这样就可以得到10个节点，
    
虚拟节点，可以解决数据倾斜的问题

## 架构推导

![703](DAD95755357C4A4B9EAF89B769F06F81)

![704](7F9CA9316EB94022808019E88FD44B2A)

注意上面的是拓扑图，实际的物理连接是什么？


模型1：多台客户端访问两台redis，**应用层服务，需要经过7层网络协议**，==对client来说，redis的连接成本很高==

---

模型2：这时就会想到代理服务器，**代理服务器是连接接入层，只负责连接，不负责数据存储、处理**

nginx是反向代理服务器，可以做负载均衡

这个模型中，==客户端的中的逻辑实现（modula、random、kemata）就转移到代理服务器中了==，这时候==只需要关注代理服务器的性能==就可以了

==代理层的无状态==：没有数据库，不负责存储数据

有了无状态，才可以一变多，谁挂谁启动变的很方便

但是代理服务器存在单点故障问题，引发模型3

---

模型3：多台代理服务器，LVS做主备集群，加keepalived

keepalived负责LVS的高可用，也间接监测proxy的健康状态

redis是单线程/进程的，不要引入太多的技术，费劲了

提到LVS，就要想到之前提过的：**无论企业后面技术多么复杂，对于客户端都是透明的**


无论后端多么复杂，只有一个vip地址；有了这个地址，无论前面有多少应用只需要记住这个地址就可以了，客户端的代码就会极其简单，不需要注册中心，发现中心去发现后面有几个server，调用的时候怎么负载也不需要关心



## twemproxy

上面提到的知识点，可以通过访问https://github.com/twitter/twemproxy进行学习，让知识点更加牢固

    readme摘抄：
    twemproxy ("two-em-proxy")

    Configuration：
        hash: The name of the hash function. Possible values are: //hash
        one_at_a_time
        md5   //md5
        crc16  //crc 16
        crc32 (crc32 implementation compatible with libmemcached)
        crc32a (correct crc32 implementation as per the spec)
        fnv1_64  //fnv
        fnv1a_64
        fnv1_32
        fnv1a_32
        hsieh
        murmur
        jenkins
        hash_tag: A two character string that specifies the part of the key used for hashing. Eg "{}" or "$$". Hash tag enable mapping different keys to the same server as long as the part of the key within the tag is the same.
        distribution: The key distribution mode. Possible values are:
        ketama  //ketama
        modula  //modula
        random  //random

## 作业：搜一致性哈希

## 集群代理

redis集群的代理模式有：twitter、predixy、cluster

国产:codis(豌豆荚，染指过源码)


## 弊端：3个模式不能做数据库用

原因就是在分布式情况下，增加机器会有问题；

modula：模数是固定的，增加机器会导致全局洗牌

random：不知道存在哪个redis上

kemata:也会导致少量的数据访问不到

### 解决方案：预分区

![705](07AFB0FF5B3F474792807328EC2F00B9)

客户端和代理服务器中有算法，如果提前设置取模数是10，那么模数值的范围：0，1,2，3，。。。。9

redis1 分配了0.1.2.3.4槽位， redis2分配了5.6.7.8.9 槽位

客户端放入数据时，先将数据hash值按10取模，得到模数值，根据预先分配好的槽位把数据放入对应的redis中

客户端访问redis时，也**hash+取模**，到相应的槽位去取数据

现在添加一台redis3，redis1和redis2把3.4 和8.9槽位分配给redis3

**并且迁移数据，迁移数据的方案就是：之前提到的RDB+AOF，先提供一份全量数据RDB，然后再提供一份增量日志AOF，实现数据追平**

==注意：预分区的取模计算是在客户端或代理层做的==

### cluster模型

![706](0FC265785C2F4F598CAF852697AF417E)

上面是客户端或代理层做的事情，在redis中连代理都省了，redis自己实现算法那

现在有三台redis，客户端没有任何算法，直接访问redis，并且都能访问

第一步：用户访问一台redis2，去查询k1 ; 


第二步：redis2 根据key的hash + 取模得到模数值，去对比自己有没有这个槽位，如果有返回数据；

如果没有，则查询这个槽位对应的redis机器，并给客户端返回，你去找redis3


第三步：客户端重新请求redis3，redis3收到请求，也是hash+取模，发现自己拥有槽位，则返回数据

这就是==cluster模型，无主模型==

redis的作者很细腻，计算向数据移动，不是数据向计算移动

做了取舍，把影响性能的东西拿掉了

## 数据分治，聚合操作、事务很难实现 -》hash tag

数据分治以后，因为数据存放在不同的机器上，对数据做聚合操作(交并差)、事务就很难实现了

把解决交给使用人员，可以使用==hash tag==来解决，现在有一组数据{oo}k1、{oo}k2,会根据tag(也就是说oo)进行 取模，这样数据就会落到一台机器上，可以就可以进行事务控制以及聚合操作

## 作业：redis官网-分区、redis集群cluster

redis中文官网--文档--分区(分区：怎样将数据分布到多个redis实例)：介绍了分区的理念

自己的实现：redis集群cluster

细则，自己看官网文档；英文文档更新，中文文档稍旧

## 作业2：predixy

百度搜索文章：predixy：一款吊打众对手的redis代理，你喜欢吗？(https://blog.csdn.net/rebaic/article/details/76384028)

## 实验：twproxy作为代理服务器

    cd /soft
    mkdir twemproxy
    
    git clone git@github.com:twitter/twemproxy.git #报错，因为系统版本低
    
    yum update nss  #升级，解决报错问题
    
    git clone  git@github.com:twitter/twemproxy.git
    
    cd /twemproxy
    yum install automake libtool  #redeme中提到要先按照这两个工具
        #autoreconf -fvi && ./configure needs automake and libtool to be installed
    
    autoreconf -fvi   #直接回车，报错：autoconf version 2.64 or higher is required
    
    ## autoconf版本低问题解决--start
    yum search autoconf  #yum从仓库安装，可能系统自带仓库的版本低
    
    # 高级仓库  mirrors.aliyun.com  epel仓库
    #  EPEL (Extra Packages for Enterprise Linux), 是由 Fedora Special Interest Group 维护的 Enterprise Linux（RHEL、CentOS）中经常用到的包。
    
    cd /etc/yum.repos.d/ 
    
    wget -0 /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo   # 下载新repo 到/etc/yum.repos.d/  多了epel.repo
    
    vi epel.repo   #多了一个仓库
    
    yum clean all #清空所有的镜像
    
    cd  #回到根目录
    cd sort/twemproxy
    
    yum search autoconf  #yum搜索发现版本库多了autoconf268版本
    yum install autoconf268 
    ## autoconf版本低的问题解决了--end
    
    
    autoreconf268 -fvi  #执行完多了configure
    
    ./configure
    
    make
    cd /src  #看到nutcracker
    #安装编译完毕
    
    #制作系统服务
    cd nutcracker/scripts
    vi nutcracker.init
        #配置文件内容
        chkconfig -55 45 #级别
        
        USER="nobody"
        OPTIONS="-d -c /etc/nutcracker/nutcracker.yml" #-d后台  -c配置文件
        
        prog="nutcracker"  #程序,如果没有指定路径，就是环境变量path基本目录，root/bin   root/sbin  root/usr/bin
        
        #函数
        start (){
            
            daemon --user ${USER} ${prog} $OPTIONS -t >/dev/null 2>&1  #daemon说明是一个服务
            
        }
    
        stop(){
            
        }
    
    
    cp nutcracker.init /ect/init.d/twemproxy #想让操作系统可以使用twemproxy的话，需要让其变成服务
    
    cd /etc/init.d/  #tewmproxy不是绿色，现在不能执行，需要更改权限
    
    chmod +x twemproxy  #变绿，变成可执行
    
    vi twemproxy  #里面start函数，提到配置文件为/etc/nutcracker/nutcracker.yml  需要准备好这个配置文件
    
    #准备nutcracker.yml配置文件
    # 再开一个标签页
    mkdir /etc/nutcracker
    cd soft/twemproxy/twemproxy/conf  //提供了配置文件nutcracker.yml
    cp ./*  /etc/nutcracker/
    cd /etc/nutcracker/
    
    # 修改nutcracker.yml配置文件
    cd /ect/nutcracker/
    cp nutcracker.yml nutcracker.yml.bak
    vi nutcracker.yml
        //看不懂，去官网看redeme
        只保留alpha的配置，剩下的不需要
        配置看图片707.png
![707](5665548114CB41DD89C04D16DBD69A93)
    
    #将可执行程序nutcracker拷贝到/usr/bin
    cd soft/twemproxy/twemproxy/src   #看到了可执行程序nutcracker
    cp nutcracker /usr/bin  #拷贝可执行程序到bin目录下，然后就可以在任何位置使用nutcracker命令了
    
    ## 到此，twemproxy代理服务器的系统服务弄好了
    
        
    # 手动启动两台redis
    mkdir data
    cd data
    mkdir 6379
    cd 6379
    redis-server --port 6379
    
    cd /data
    mkdir 6380
    cd 6380
    redis-server --port 6380
    redid-cli -p 6380 shutdowm #不确定redis是否已经启动，可以先停再启动
    redis-server --port 6380
    
    # 启动twemproxy代理服务器
    service twemproxy start
    
    redis-cli -p 22121 //通过代理连接redis客户端
    
    #正常的插入、访问
    
    #缺点：事务、聚合、
    keys *    #Error:Server closed the connection
    watch k1
    multi 
    
==这些事情应该是运维执行，如果运维告诉你一些命令(keys * /watch/multi)不可用，那你就可以知道使用了代理或集群==
    
    
## 实验：Predixy    

    C++编译器不达标
    
    
    github--releases
    release1.0.5  编译后的压缩包
    
    
    cd 
    
    cd soft/
    mkdir predixy
    cd predixy
    wget xxxx
    tar xf predixy-1.0.5-bin-amd64-linux.tar.gz
    cd predixy-1.0.5
    cd bin/   #有可执行程序：predixy
    ll -h   #  predixy文件大小为14M
    
    cd ../conf   #配置文件--怎么用看readme---详细文档
    
    #redis实例配置部分
        Distribution modula|random
        Sentinels {
            + addr
            ...
        }
        Group xxx {
            [+ addr]
            ...
        }
    # 可以利用哨兵把数据打散到两台主从复制里
    
    vi prodixy.conf
        #完美继承redis的配置格式，自描述的配置文件
        Bind 127.0.0.1:7617
        
        ##########SERVERS###########
        Include sentinel.conf
    
    vi sentinel.conf
        
       #小技巧 
        :.,$y    # 复制光标到最后一行的内容 shift+: 开启末行模式    .光标位置   $最后一样  y复制
        G   #粘贴
        :.,$s/#//   #删除前面的#   shift+:    .  $   s替换   /#替换的内容   //空白
         
        配置内容图片708.png
![708](42EA2B4A43484E3C857CCE3759D66A48)
        
        Group shading001  # 对应哨兵配置中监控的redis实例的名称
    
    #修改哨兵配置文件
    # 再开一个标签
    cd test
    vi 26379.conf
        port 26379
        sentinel monitor ooxx 127.0.0.1 36379 2   #36379主   36380从  逻辑名称ooxx ,跟Group一一对应
        sentinel monitor xxoo 127.0.0.1 46379 2  #46379主  46380从
    
    vi 26380.conf
        port 26380
        sentinel monitor ooxx 127.0.0.1 36379 2   #36379主   36380从 
        sentinel monitor xxoo 127.0.0.1 46379 2  #46379主  46380从
        
    vi 26381.conf
        port 26381
        sentinel monitor ooxx 127.0.0.1 36379 2   #36379主   36381从 
        sentinel monitor xxoo 127.0.0.1 46379 2  #46379主  46381从
        
        
    redis-server 26379.conf --sentinel
    redis-server 26380.conf --sentinel
    redis-server 26381.conf --sentinel
    
    
    #启动redis主从
    cd test
    mkdir 36379
    mkdir 36390
    mkdir 46379
    mkdir 46390
    
    
    cd 36379
    redis-server --port 36379
    cd 36380
    redis-server --port 36380 --replicaof 127.0.0.1 36379
    
    cd 46379
    redis-server --port 46379
    cd 46380
    redis-server --port 46380 --replicaof 127.0.0.1 46379
    
    #启动predixy代理
    cd soft/predixy/predixy-1.0.5/bin
    ./predixy ../conf/redixy.conf
        #启动日志709.png
![709](3A5265B046E34A9EA48E7E4CD575CA6B)
    
    #连接客户端
    redis-cli -p 7617
    
    set k1  xvc
    set k2  xfdsfdsafa
    
    redis-cli -p 36379  //keys *  有k1
    redis-cli -p 46379  //keys *  有k2
    #打散了数据
    
    
    set {oo}k1  fafa
    set {oo}k2  fafdafadfasdfa 
    redis-cli -p 36379  //keys *  有k1
    redis-cli -p 46379  //keys *  有k2  {oo}k1   {oo}k2
    #通过人为的设置，将数据存储在一台机器上
    
    
    WATCH {oo}k1   //ERR forbid transaction in current server pool   因为代理后面有两套主从复制，数据分治了
    
    
## predixy 支持事务

    vi sentinel.conf
        #删除group xxoo,只保留了一个主从复制
        配置文件图片710.png
![710](F79DFF0B33554DCA9D984AD363BDB753)
    
    cd ../bin
    ./predixy ../conf/predixy.conf
        #启动日志711.png（图片不对）

        
    redis-cli -p 7617
    set k1  fdafafa
    set k2  vsfaffsfaf
    #数据只会存储在36379
    
    WATCH k1
    MULTI
    get k1
    set k2 22222
    exec
    
    #主36379挂掉，哨兵会让36380升级为主，客户端访问数据是无感的
    
代理解耦后端的复杂度，前端使用不需要关心后端

## 实验：redis-cluster

==注意：开启redis集群，redis配置文件中要设置：cluster-enabled yes，开启cluster模式，否则无法开启集群模式==




redis可以组建一个无主模型的多节点cluster，且每个主机/实例要认领一些槽位，涉及到启动几台？认领哪些槽位？如果一台挂了，数据会丢失吗？reids的哨兵是不是可以在实例中挂一个层？这些问题有一套现成的脚本，在src/create-cluster
    
    
    老版本的脚本需要安装rb
    新版本不需要了，已经集成了
    
    cd soft/redis-5.0.5/utils/create-cluster   #脚本：create-cluster
    vi README
        To create a cluster,follow thes steps
        1.Edit create-cluster and change the start / end port,depending on the number of instances you want to create;
        2.use "./create-cluster start" in order to run the instances.
        3.Use "./create-cluster create" in order to execute redis-cli --cluster create ,so that an actual redis cluster will be created;
        4.now you are ready to play with the cluster.AOF files and logs for each instances are created in the current directory.
        
        In order to stop a cluster:
        1. use "./create-cluster stop" to stop all the instances.After you stopped the instances you can use './create-cluster start' to restart them if you change your mind.
        2.user "./create-cluster clean" to remove all the AOF / log files to restart with a clean environment.
        
    
    vi create-cluster
        NODES=6   #redis实例个数
        REPLICA=1  #副本个数：1个，在6个实例时，就是3主3从
        
        
    ./create-cluster start  //会帮助启动6个实例，端口为30001到30006
    ./create-cluster create //分赃
        #执行结果：图片711.png
![711](CD61E49BC74646E9AB22410CF0AB2C1D)
    
    redis-cli -p 30001
    set k1 afafafaf  #(error) MOVED 12706 127.0.0.1:30003   以error报错，通知需要在30003才能完成操作
    
    redis-cli -c -p 30001 
    set k1 fafdafafa  #redirected to  slot  [12706]  located at 127.0.0.1:30003  这时候连接已经跳转到30003
    get k1    # 127.0.0.1:30003>  代表已经连接到30003
    
    #事务的执行错误
    #执行结果，图片712.png
![712](1C38BF09995647D3B81739E9CAA844BE)
    
    #事务执行成功 
    #执行结果，图片713.png
![713](4EAA2A691C154ECA81184A44F3E7AB57)
    
    ./create-cluster stop
    ./create-cluster clean
    
    
    # 手工配置cluster集群
    ./create-cluster start
    ./create-cluster create xxxxx
        #图片714.png
![714](BCB27518B1DE4093A5985FDD2BB86767)
    
    redis-cli --cluster help #查看帮助
    
    # reshard
        图片715.png
        图片716.png
        图片717.png
![715](3D45B4E3E61041168275213B44D93D38)
![716](799F537B751A43E587B5C56B0E09DEAC)
![717](C6C4DC2C495047F7A4EEB4ED8A365D79)

        
    redis-cli --cluster info 127.0.0.1:30001
        图片718.png
 ![718](6DCCC66E029F429882F9FD78F9C89D59)
    
    redis-cli --cluster check 127.0.0.1:30001
        图片719.png
![719](787DCAF76C6D4CD187A1E745A987B3DC)
    
### ==reshard  解决数据移动、数据倾斜的问题、新增节点、删除节点；集群模式给了灵活的方案   、只能手动操作==
    
想必须移动数据最多的槽位。或者指定的槽位，是做不到的，别较真，redis作者连多线程不想给你实现，是一个懒的人

    
    
**注意：理论知识是为了面试，实操部分是为了将来入职以后试用期通过**

如果课程内容听一遍就能听懂，那么这个钱就白花了

课下一定要多实操、多练习

**不要照着实操的记录敲命令，一定要参与设计、站在上帝的视角来设计，来理解理论知识**

**现在面试：redis、netty是最值钱的，springcloud大把的人会**



