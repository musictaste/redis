[TOC]

# V-list

![301](40CD38403DF04A0DB967420A23FB09C5)

链表：单向链表、双向链表；有环的，无环的

redis的key中有head、tail头尾指针属性
 
## 同向命令-栈 & 反向命令-队列 
   
    head @list  //以L开头 代表左边的，以R开头 代表右边的，以L开头 代表List类型
    
    lpush k1 a b c d e f    //从左边添加abcdef元素，在内存中的顺序应该为fedcba
    
    rpush k2 a b c d e f //从右边添加元素，在内存中的顺序为abcdef
    
    lpop k1 // f 弹出一个元素, 后进先出--栈
    
    rpop k2 // f 弹出一个元素
    
    rpop k1 // a
    
    rpop k2 // a
    


##  list也有正负索引
    
    FLUSHALL
    lpush k1 a b c d e f
    LRANGE k1  0 -1 // 取出所有元素fedcba  同向   list也有正负索引
    
    rpush k2 a b c d e f
    LRANGE k2 0 -1  // abcdef  反向
    
    
    LINDEX k1 2  //d 下标为2
    LINDEX k1 -1  // a  最后一个元素
    
    LSET k1 3 xxxx  //下标3设置为 xxxx
    
    LRANGE k1 0 -1 //
    
## 数组：对索引操作的话

    lpush k3 1 a 2 b 3 a 4 c 5 a 6 d 
    
    LRANGE k3 0 -1  //得到d6a5c4a3b2a1
    
    LREM k3 2 a //得到d65c43b2a1  从左移除 count=2 value=a 
        //为正数，从左开始移除
        //为0
        
    LINSERT k3 after 6 a //元素6后面插入a  d6a5c43b2a1 ;如果有两个6，只能是从左的第一个6后面插入a
    
    LINSERT k3 before 3 a //元素3前面插入a  d6a5c43ab2a1 ？？？？
    
    lrem k3 -2 a //从左移除后2个 得到d6a5c43b21
    
## 阻塞，单播队列  FIFO
    
    B开头的  阻塞的弹出元素
    
    实验环境：开启三个窗口连接redis客户端（redis-cli）
    
    第一个窗口：
    keys * // 键值对为空
    blpop ooxx 0 // timeout=0 一直阻塞
    
    第二个窗口：
    blpop ooxx 0
    
    
    第三个窗口：
    
    rpush ooxx hello   //插入数据后，第一个客户端退出阻塞，并且打印hello；但第二个客户端没有退出阻塞
    
    rpush ooxx world //第二个窗口退出阻塞，并打印world
    
阻塞，单播队列  FIFO

==埋个伏笔，redis是支持消息订阅的==

## ltrim 删除start、end两端的元素

    help ltrim //删除start、end两端的元素
    
    lpush k4 a b c d e f d fs sdf sdf sdf sd f
    
    LTRIM k4 0 -1 //删除多少个元素？0个
    
    LTRIM k4 2 -2 //结果为：  sdf sdf sdf  fs d f d e d c  b 
    
# V-hash
    
类似java的hashmap

    set sean::name 'limiao'
    
    get sean::name
    
    set sean:age 18
    
    keys sean*  //得到name/age
    
思考：如果一个用户有100个字段，采用上面的方式，增加了IO成本；如果有键值对就好了，这样就引入了hash的概念

    help @hash
    
    hset sean name limiao
    
    hmset sean age 18 address bj
    
    hget sean name  //limiao
    
    hmget  sean name age //limiao 18
    
    hkeys  //获取keys
    
    HVALS  //获取values
    
    HGETALL //获取全部的keys values
    
    HINCRBYFLOAT sean age 0.5  //加0.5
    
    HINCRBYFLOAT sean age -1 //减1，减法通过加法实现；注意：HINCRBYFLOAT浮点型
    
## 应用场景：点赞、收藏、详情页

场景：1.点赞，收藏（对field进行数值计算）

2.详情页（一次取出商品的相关字段，减少交互次数）
    
    
# V-set

![302](53471C3DD4AC489B80DDED95A6E8FD8C)
    
==【无序==】&&【随机性】

放入的多少不同，元素存储的顺序不同

==去重==

==使用Redis时，虽然Redis提供了很多的功能(API)，但是要清==楚哪些方法会影响性能，例如SMEMbers 会消耗redis所在主机网卡的吞吐量

==如果真的需要做这样的功能(需要获取所有的keys)，可以单拿出几台服务器放这样的几个集合；单切出来提供这样的功能，不要和其他的key-value功能混合使用==

## 增删改查

    help @set
    
    FLUSHDB
    
    sadd k1 tom sean peter ooxx tom xxoo  //Tom去重
    
    SMEMBERS k1
    
    srem k1 ooxx xxoo
    
## 集合操作-交并差集（使用比较多）

    sadd k2 1 2 3 4 5
    sadd k3 4 5 6 7 8
    
    SINTER k2 k3 //直接求交集，并打印,结果=4.5
    
    SINTERSTORE dest k2 k3 //将交集保存到dest中
    
    SMEMBERS dest //结果=4.5
    
==带store，可以看出作者的细腻；求交集并生成key的过程是在服务端一步完成的，数据没有进入IO；==

==不仅要知道这两个命令，还要多说两句，这样就可以让面试官知道你考虑技术选型的问题==
    
    SUNION k2 k3  //并集
    
    SDIFF k2 k3  //差集：1，2，3
    
    SDIFF k3 k2 //差集：6.7.8
    
差集是有方向的，作者并没有做过多的设计，
    
    
## 随机事件 SRANDMEMBER SPOP

SRANDMEMBER key count //count也分正、负、0三种情况

正数：取出一个去重的结果集（不能超过已有集）

负数：取出一个带重复的结果集，一定满足你要的数量

如果：0，不返回

    
### 应用场景

1.抽奖：

    有10个奖品
    用户数量：  <10   >10
    用户中奖：是否重复
    
    FLUSHALL
    
    sadd k1 tom ooxx xxoo xoxo oxox xoox oxxo  //应该是用户，不是奖品
    
    //抽奖抽了三回，每一回抽奖，一个人最多中一件礼物
    SRANDMEMBER k1 3  
    SRANDMEMBER k1 3
    SRANDMEMBER k1 3
    
    //中奖人数可以重复
    SRANDMEMBER k1 -3
    SRANDMEMBER k1 -3
    SRANDMEMBER k1 -3
    
    //抽奖的场景：礼品多(购物卡)，用户少，想要每个人都多抽几张购物卡，并且领导要尽量抽到大面额的购物卡
    
    //使用redis满足不了“领导尽量抽到大面额的购物卡”，违反了公平公正的原则
    
    //场景1：用户小于礼品数
    SRANDMEMBER k1 -20  //一个用户多抽中几次
    
    //场景2：用户大于礼品书(公司年会抽奖，中奖以后，以后就不能再参与抽奖)
    SPOP k1

2.解决家庭争斗！

# V-Sorted_Set（最难的，最容易懵）

![305](13EF7A69C7D8432CA2118D7ED2ECB0DA)


==Sorted_Set：去重+有序（这个有序是指排序），不同于Set中的有序（存储的顺序）==

Sorted_set是最难理解的，也是最容易懵的

![303](BF5E08B4C8B74E09B8DC298A5775B6E4)

## 在讲Sorted_set之前，先铺垫一下；

有苹果、香蕉、梨，想让他们排序，按照什么规则排序？名称、含糖量（注意是隐藏的，用户看不见）、大小、价格、吃货热度

==Sorted-set 是有分值的，如果不给出分值，谁都不知道该怎么排序==

如果分值都为1，那么默认按名称进行排序

==还有正负向索引==

## 物理内存是左小右大,不随命令发生变化 zrange zrevrang（面试坑）

==注意：物理内存是左小右大,不随命令发生变化，这块面试官容易在这里挖坑，或往坑里带==

    因为S已经被Set占用了，所以用Z来代替

    FLUSHALL
    zadd k1 8 apple 2 banana 3 orange //物理内存是左小右大

    ZRANGE k1 0 -1  //取出所有 结果为；banana orange apple
    
    ZRANGE k1 0 -1 withscores //带分值的取出所有
    
    ZRANGEBYSCORE k1 3 8 //取分值3到8  结果为：orange apple
    
    ZRANGE k1 0 1 //按照分值取前两个
    
    ZREVRANGE k1 0 1 //从高到低取前两个  apple orange 
    
    ZRANGE k1 -2 -1 //跟ZREVRANGE结果不一样，orange apple
    
    ZRANGE k1 -1 -2 //结果：empty list or set
    
    ZSCORE k1 apple //8   根据名称得到分值
    
    ZRANK k1 apple //2  根据名称得到排名
    
## 数值计算

    ZRANGE k1 0 -1 withscores //取出所有，带分值
    
    ZINCYBY k1 2.5 banana  //banana分值增加2.5
    
    ZRANGE k1 0 -1 withscores //排序发生变化，
    
排序发生变化，==应用：歌曲排行榜，热度（下载、播放），倒序==

==通过redis就可以使用歌曲排行榜，不需要mysql，因为mysql在并发下会触发事务==
    
## 集合操作：交并集

不同于Set的集合操作，关注的是分值的交并差，注意还有权重 和 聚合指令（对分值求和、最小、最大）

    ZUNIONSTORE destination numkeys key [key...] [weights weight] [aggregate sum|min|max]
    
    FLUSHALL
    
    zadd k1 80 tom 60 sean 70 baby //考试成绩
    
    zadd k2 60 tom 100 sean 40 yiming
    
    ZUNIONSTORE unkey 2 k1 k2 //不设置权重、计算，默认为sum计算
    
    ZRANGE unkey 0 -1 withscores //结果为yiming 40   baby 70   tom 140  sean 160  
    
    ZUNIONSTORE unkey1 2 k1 k2 weights 1 0.5 //设置权重1,0.5、默认为求和
    
    ZRANGE unkey1 0 -1 withscores //yiming 20    baby 70    sean 110   tom 110
    
    ZUNIONSTORE unkey1 2 k1 k2 aggregate max
    
    zrange unkey1 0 -1 withscores //yiming 40  baby 70   tom 80  sean 100
    
    
## 排序是怎么实现的？增删改查的速度？-跳表（面试常考）

![304](55E77AFCEDE543C9945348BEF21EAE04)

### 简单说一下跳表

==简单说一下跳表，只是通俗讲解，逻辑并不严谨，主要是做启蒙==

有一个链表，进来一个元素，有序的话，应该怎么实现的呢？

最简单的方式就是：从左到右依次遍历，比较大小，然后插入

上面的方式有个问题：如果链表很长，有1万个元素，时间复杂度为O(n), 时间复杂度就变高了

跳表：链表中的元素不仅保存前后节点的指针，还保存了指向上一层的指针

跳表可以有很多层，如果没有就指向null

查找的过程

    当一个元素11插入的话，找到头节点3，找到头节点的最高层，比较；如果小，该值为头节点；如果大，发现后面没有元素了，进入下一层
    
    在第二层，比3 大，比22 小；后面的不执行了，然后到达第三层()找到22
    
    在第三层：11跟22比较，发现小，往前跟7比较，大于7，则插在7后面
    
==插入的过程会随机造层==

==跳表就是牺牲存储空间，换查询速度==

越往上，层里出现的点越少，类似树的遍历

插入的过程    

    假设插入元素11的时候，要造3层的，
    先向前查找，哪个元素指向上一层，遍历发现元素3，
    到达第二层，把原先3指向22中间插入11；
    因为要造3层，询问当前元素3是否还有上一层，有，则到达第三层，在元素3后面插入11
    
修改元素：就是把原先的值剔除，再重新插入

这就是跳表的增删改查的过程

跳表的另外一个名字就是：平衡树，是类 平衡树的

什么是平衡树，左旋、右旋，插入元素的复杂度比较均匀

### 跳表，增删改查综合来看，平均值相对最优