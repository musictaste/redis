[TOC]

# redis是单进程单实例的；那么并发很多请求，redis如何变得很快的呢？

==redis为什么是单进程单实例的，主要原因是：保证数据一致性，附带影响是减少了用户态内核态切换时的损耗==

了解了NIO的概念，要学以致用，现在聊Redis

一台linux服务器，有两个redis：端口号分别为6379/6380；注意：==redis是单进程(linux)、单线程(windows)、单实例的；那么并发很多请求，redis如何变得很快的呢？==

==注意：Redis不是只有一个进程、线程；而是有一个进程、线程处理 用户请求；还有别的线程，只不过不对用户请求做处理，所以我们常说redis是单进程、单线程、单实例的==

linux系统中有内核Kernel，客户端经过三次握手，到达Kernel，建立Socket；

Redis进程和Kernel之间使用的是Epoll（内核提供的系统调用）

内存是快的，IO是慢的

==数据从客户端到达内核，是有“顺序”性的，是指每个连接内的命令是顺序处理的==

指的是一个客户端 发出的命令是线性的（不管是单线程还是多线程）

一定要明白这个顺序性，因为是在分布式、多线程的场景下，数据的一致性，事务是最头疼的

==开发的时候一定要注意这个点：一定要将对一个key的操作放到一个节点上，类似kafka==

# Redis的进阶使用

先聊Redis命令行的客户端，API的先不谈，因为Redis是原语的，只要会了命令行的指令，就可以在API中找到对应的调用

## 基本命令

    ps -fe | grep redis //redis进程已经启动，端口为6379/6380
    
    redis-cli  //启动redis客户端，默认为6379  exit退出
    
    redis-cli -h //查看帮助手册
    
    redis默认有16个库
    
    redis-cli -p 6380  //连接6380
    
    set k380:1 hellp  //添加key-value
    
    get k380:1  //根据key，获取value
    
    select 8  //查询8号库，默认为0号库
    
    exit
    
    redis-cli -p 6380 -n 8   //访问6380端口的8号库
    
    help   //redis客户端的帮助手册，需要进入redis客户端；help @<group>  help <command>  help <tab>  quit
    
    -----通用组
    
    help @generic  //通用组；DEL   EXISTS   EXPIRE   KEYS   MOVE  OBJECT  PERSIST  TYPE
    
    keys *   //所有建过的key
    
    FLUSHDB  //清库；尽量不要使用；FLUSHDB只有，再执行keys *  就没有keys的结果了；还有FLUSHALL
    
    -----基本类型
    help @string  //list  set  sorted_set  hash
    
    help @pubsub  //pubsub消息队列  transactions ....
    
### **V-字符串string** 
    
String类型，Byte ，具体有字符串、数值、bitmap（位图）

#### 面向字符串的基本操作

    set k1 hello
    
    get k1
    
    help set  //set key value 有效期 nx|xx  ;
        //nx:**key不存在的时候才会设置，只能创建**；使用场景：分布式锁，谁成功谁拿到锁，其他人失败
        //xx:**key存在的时候才会操作，只能更新**
        
    set k1 ooxx nx //返回nil
    
    set k2 hello xx //返回nil
    
    mset k3 a k4 b //
    
    get k3
    
    mget k3 k4
    
    //还有APPEND    GETRANGE   GETSET取回老的值，设置新的值  SETRANGE
    
    APPEND k1 " world" //值=hello world
    
    GETRANGE k1 6 10  //得到world
        //正反向索引：有一系列的元素，正向开始为0，依次+1；逆向从-1开始，依次-1
        
    GETRANGE k1 6 -1 //得到world
    
    getRANGE k1 0 -1 //得到hello world
    
    SETRANGE k1 6 mashibing //变成hello mashibing
    
    STRLEN k1 //长度
    
## 高级使用

### type

**用户访问的时候，直接返回type类型，看这个方法是否具备操作，可以快速返回；不需要真的去取value的值，参与计算报异常，可以规避异常**

    type k1  //string    key中有type：记录value的类型
    
    FLUSHALL
    
    set k1 99  //type=string
    
    type k1 
    
    set k2 hello  //type=string
    
### **V-数值incr**

    object help
    
    OBJECT encoding k2 //返回embstr   (k2的值为hello)
    
    OBJECT encoding k1 //返回int  (k1的值为99)
    
    INCR k1  //加1，值变为100
    
    help incr 
    
    INCRBY k1 22   //加22，值变为122
    
    DECR k1  //减1
    
    DECRBY k1 22 //减22
    
    INCRBYFLOAT k1 0.5  //加0.5

#### ==应用场景：抢购，秒杀，详情页，阅读量，点赞，评论==

抢购，秒杀，详情页，点赞，评论

==规避并发下，对数据库的事务操作，完全由redis内存操作代替==

==适用于不是特别精准的，不是重要的数据==

银行的钱就不能使用redis了，必须染指事务、持久化，数据可靠性必须保证

### encoding

也可以规避异常

==作者为什么要设计encoding？==

**原因：Redis会发生预判断，可以直接触发计算，不需要进行排错的过程(不需要进行编码转换的问题)，用于提速（可以把类型转化你的过程忽略掉）**

****Redis底层是按照字节存储的**，在key上面做了优化，如果没有encoding的话，每次都要判断type，看能否转换成功**

    set k3 j..jj  //添加39个字符 ,encoding=embstr
    
    OBJECT encoding k3
    
    APPEND k3 jjjjj //再添加5个,encoding=raw
    
    OBJECT encoding k3 
    
    ===============二进制安全
    FLUSHALL
    
    set k1 hello
    
    STRLEN k1 // length=5
    
    set k2 9 
    
    STRLEN k2 //length=1,  encoding=int
    
    APPEND k2 999 
    
    OBJECT encoding k2 // raw
    
    INCR k2 //可以加1 ，值=10000
    
    OBJECT encoding k2  // int 
    
    STRLEN k2  //  length=5 为什么？因为编码encoding 并没有影响数据的存储
    
    set k3 a 
    
    APPEND  k3 中
    
    STRLEN k3  //length=4 为什么？因为Xshell软件的编码采用UTF-8(3个字节)，GBK（2个字节）
    
    //将Xshell的编码方式由UTF-8切换为GBK
    
    set k4 中  //length=2
      
    exit //退出
    
    redis-cli --raw  //加raw触发编码级格式化，不加会显示anxci码，超过显示16进制
    
    get k4 // 中
    
    get k3  //发生乱码
    
    STRLEN k3 //4
    
    STRLEN k4 //2
    
    set k5 5 //会发生预判断，int
    
    INCR k5 //可以直接触发计算，不需要进行排错的过程(不需要进行编码转换的问题)，用于提速（可以把类型转化你的过程忽略掉）

### 二进制安全
    
**二进制安全： IO流有字节流、字符流；redis从Socket中拿到的流是字节流，不是字符流；所以只要未来双方的客户端有同一的编解码，数据就不会被破坏**

不同的语言对int的定义不同，所以可能会发生 **节段溢出** 的情况

类似于多语言开发时，更倾向于于使用json、xml这种文本形式进行交互，而不是用序列化(需要编解码)

==编码encoding 并没有影响数据的存储==

==Hbase也是二进制安全的==


作者为什么要设计encoding？

原因：Redis会发生预判断，可以直接触发计算，不需要进行排错的过程(不需要进行编码转换的问题)，用于提速（可以把类型转化你的过程忽略掉）

Redis底层是按照字节存储的，在key上面做了优化，如果没有encoding的话，每次都要判断type，看能否转换成功

==也就是说，在使用redis时在用户端沟通好 数据的编解码，redis里面是没有数据类型的==

### key对象中有type，encoding、length，是redis做的优化，为了高并发时极快的返回，时间复杂度基本为O(1)

### GETSET、MSETNX

    set k1 hello
    getset k1 mashibing  //返回hello，并设置了mashibing
    
**getset这个命令不是原子性，就是get +set 的组合，推出getset的原因是什么呢？就是成本问题，减少IO通信**

**MSETNX k1 a k2 b  当keys不存在的时候，设置多个值【原子操作，要么全成功，要么全失败】**

---


    MSETNX k1 a k2 b   //MSETNX  //当keys不存在的时候，设置多个值，
    
    mget k1 k2  //k1=a   k2=b
    
    MSETNX k2 c k3 d   //原子操作，要么全成功，要么全失败
    
    mget k1 k2 k3  //k1=a   k2=b


## 答疑

Redis使用NIO-Epoll,

==Mysql更倾向于使用BIO，一个连接启一个线程,为什么？因为假设有很多连接进来了，比如十万，百万，这些连接很快就会到达很多请求，如果每一个请求都要触发磁盘IO，这时磁盘的带宽就会成为瓶颈，这样的话使用Epoll没有意义，还不如直接使用BIO==

**Mysql:索引会预加载，索引的主干预加载到内存，索引的叶子还在磁盘；**、

**而且mysql可以开启缓存，一条SQL语句，如果数据不大，数据会缓存在内存中，并且这个sql语句的hash放在内存里，如果下一条语句一样，那么会从缓存直接返回，后面的语法树等等就不做了；**

听起来好像可以增强Mysql性能，但是毕竟Mysql数据是大过内存的，内存只有4G，而Mysql数据可以有1T/2T；**如果开启缓存，Mysql的行表比较多的话，这时不仅没有增强，还有可能降低Mysql性能**

**亲密度**：如果linux是4核，会启4个进程，也就是有4个核心；**启动redis，把它亲密到第3个核心上**，其他的事情在其他进程上运行；CPU是有时间片的，**这样3号CPU就一直在处理redis，也减少了一二三级缓存的清理，这样Redis才能达到1秒可以处理10万的**

## **V-bitmap位图** (Redis最值钱的)

一个字节有8个二进制位，redis有正反向索引，字节会有索引；

并且二进制位也有索引，并且二进制位的索引不是相当于字节的，而是整个一长串字节的

![201](22E2B19BEA1F418D8A87BFF167E92C66)

### setbit  设置位

    help setbit  //group:string,是二进制位，不是字节数组
    
    setbit  k1 1 1  //k1的长度=1，值为@ 0100 0000 （对应ASCII码） 
    
    STRLEN k1 // 1
    
    get k1  //@
    
    setbit k1 7 1 //k1的长度=1，值为A（对应ASCII码）  0100 0001=41=A
    
    man ASCII  //查看ASCII码

    setbit k1 9 1  //** k1的长度=2，值为A@（对应ASCII码）0100 0001 0100 0000 =A @
    
### 补一个常识：字符集 ascii码

    字符集是ascii码；其他一般叫做扩展字符集
    那什么是扩展呢?扩展：  其他字符集不在对ascii重编码
    ASCII的规则是0xxxxxxx，第一位必须是0
    你自己写一个程序，字节流读取，每字节判断时，如果第一位是0，则按ASCII处理，其他的处理方式(没记清楚)
    
    这也就是为什么会有乱码，英文字母不会有乱码，但是中文会有乱码，因为ASCII中没有中文对应的字符集，如果不指定编码格式，那么GBK的编码在UTF-8就是乱码
    
### bitpos  位下标

注意start，end指的是字节下标，不是二进制下标

注意返回的二进制位的下标，不是字节中的二进制下标

    help bitpos //find first bit set or clear in  a string
    
    bitpos k1 1 0 0 // 1  第一个字节：0100 0001 注意start，end指的是字节下标，不是二进制下标
    
    bitpos k1 1 1 1   // 9 第二个字节：）0100 0001 | 0100 0000；注意返回的二进制位的下标，不是字节中的二进制下标
    
    bitpos k1 1 0 1 //1  返回第一个字节出现的问题
    
### bitcount  位统计

注意start，end指的是字节下标，不是二进制下标

    bitcount k1 0 1 //3  0100 0001 0100 0000 注意start，end指的是字节下标，不是二进制下标
    
    bitcount k1 0 0 //2
    
    bitcount k1 1 1 //1
    
### bitop 位运算

    FLUSHALL
    
    setbit k1 1 1
    setbit k1 7 1
    get k1  // A 0100 0001
    
    
    setbit k2 1 1
    setbit k2 6 1 
    get k2 // B 0100 0010
    
    bitop and andkey   k1  k2 
    get andkey  //@ 0100 0000
    
    bitop or orkey k1 k2
    get orkey //C 0100 0011 
    
### ==应用（面试题，Redis必考）==

应用1：有用户系统，统计用户登录天数，且窗口随机（统计范围随机）

**如果是Mysql，需要创建一张临时表，字段：用户id(4个字节)、日期（4个字节），也就是一个用户的一个登录需要8个字节；用户量大时，成本很高**

==Redis方案：一年366天，366个二进制位，46个字节；也就是用46个字节可以记录用户一年365天的登录状态；==，==优势：存储小，计算还快==

    setbit limiao 1 1  //第二天登录
    setbit limiao 7 1  //第八天登录
    setbit limiao 364 1   //第365天登录
    STRLEN limiao  // 共46个字节
    BITCOUNT limiao -2 -1  //统计最后两周,最后两个字节代表两周
    
            01  02   03  04
    sean    0    1   0   1   010101
    json    0    1   0   1   011111

    每用户46B *  用户数 1000 0000  =460 000 000

应用2：京东618做活动：登录送礼物,大库备货多少礼物?

    假设京东有2E(亿)用户,大库备货多少礼物?
    
    僵尸用户
    冷热用户/忠诚用户
    
    活跃用户统计！随即窗口
    比如说 1号~3号统计活跃用户： 有过登录    需要去重

    方案：二进制位旋转，以日期作为key    二进制是用户id，即用户1的下标为0

    setbit 20190101   1  1  //用户2登录 用户李淼的二进制位为1 
    setbit 20190102   1  1  //用户2登录
    setbit 20190102   7  1  //用户8登录
    bitop  or   destkey 20190101  20190102  //这两天的用户，做或运算
    BITCOUNT  destkey  0 -1 //统计0到最后一位中，值为1的数量
    
                u1   u2   u3 
    20190101    0     1   0    000100
    20190102    1     1   0    10101
 
![202](8941A7DB57AB4BA1A73551983FCEE658)
   
# 作业：面试准备：https://db-engines.com/en/articles中查看 数据模型

https://db-engines.com/en/articles

查看数据模型(K-v, 列式、文档、关系型、图、事件、时序、搜索)

了解并背会数据模型，差异在哪，在面试中让面试官了解自己的水平