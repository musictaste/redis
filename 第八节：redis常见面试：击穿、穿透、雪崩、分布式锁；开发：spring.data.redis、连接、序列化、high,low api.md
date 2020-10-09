[TOC]

# 课程前导

redis初中级的使用，高级的使用先不讲

**spring中默认已经是luttce，jedis不再是默认**

**jedis生存空间越来越小，是线程不安全的，虽然底层也是poll；在大数据中会用到jedis，因为在大数据中所有的任务，例如spark的任务就是一个task，是一个进程，去访问redis时，不需要容器、spring、连接池，就是连接redis取数**

**luttce底层是netty**

# 面试常问

## 击穿

![801](25EEB69FB5394AC384B22ED4F95809D9)

击穿和穿透、雪崩不是一回事

互联网用户也就是client，有10万并发或亿级流量，通过nginx、lvs过滤到三四万的水分，剩下三四万的流量到达后端的server服务，这些服务会组成微服务的群体

server客户端访问redis，而redis我们一般用作缓存，不是数据库；缓存中一般会存放热数据，那么redis后面还有数据库DB，存放业务数据

**后面会有亿级流量下多级缓存，利用nginx规避流量，拦截掉三四万的水分，达到微服务群体中的流量就剩三四万，如果对数据有请求的话，利用redis再规避掉两三万，到达数据库的流量就更少了，那么数据库就能hold**

前面10万的用户并发达到数据库只有几百或几千，这才是一个架构师横看一个项目时应该做的事情

**我们用redis做缓存，会做两件事情，一是给key设置过期时间，二是采用LRU、LFU将冷数据清理掉，节省空间留给热数据**

redis中的一些数据曾经有，因为长时间没有使用，通过过期时间或LRU、LFU清理掉了，清理掉以后，有请求来访问了，这时候就会绕过redis到达数据库，就相当于在redis中穿了一个孔，这就是==击穿，就是原先reids有这部分数据，因为过期时间或数据变冷，导致redis把数据清理掉了，再有请求访问时，绕过redis到达数据库==

==击穿的原因就是：1.前面肯定发生了高并发，2.KEY的过期（给KEY设置过期时间或LRU、LFU），造成并发访问数据库就==

如果只有两三个请求访问过来，并不会对数据库造成高并发

### 如何解决击穿 -》 分布式锁 -》 zookeeper

解决这个问题，首先要知道**1.redis中现在没有key**；redis是缓存，那么内存不会太大，另外key始终都会过期，如论是设置过期时间还是LRU、LFU算法，**2.有高并发**

==那么解决方案就是：1.阻止高并发到达DB；2.redis是单进程单实例的，那么就可以利用setnx(nx:没有的时候才会set)，setnx就相当于一把锁==

那么解决方案的流程：

服务server的高并发请求 **第一步：很多请求访问redis，发现没有**(get key发现为null)

**第二步：很多请求抢一把锁setnx**(setnx,只有一个请求能获得锁)

**第三部：只有获得锁的请求才能访问DB**（获得锁的请求去访问DB；没有获得锁的请求，sleep睡眠一会；睡眠时间到了以后再次执行步骤1，如果这是第一个请求还没有释放锁，那么get key还是null,执行第二步获得锁，肯定不成功，则进行第三步睡眠；睡眠时间到了，再执行步骤1，这时如果第一个请求已经释放锁，并将key缓存在redis中，那么get key就会获得key，直接返回key）

这个步骤有什么问题呢？

**问题1：如果获得锁的连接挂了？没有释放锁，造成了死锁；**

**解决方法：可以设置锁的过期时间**

**问题2：如果获得锁的连接没有挂，但是锁超时了**（没有取回数据，但是已经过期了），造成了连接不停的抢占锁，但是都没有在过期时间内从数据库取回数据

**解决办法：采用多线程，一个线程去DB读取数据；一个线程监控数据是否已经取回来，没有取回来，则延迟锁的过期时间**

==这就相当于server客户端要自己实现分布式协调，会是客户端代码变复杂==

**技术推导这就可以了，接下来就是分布式锁的引入；也就是zookeeper的优点**

## 穿透

![802](0DB6A461E01748F7AC60E5C399A777FB)

前提：redis用作缓存

==穿透的原因：业务访问的数据在系统中根本就不存在==，**造成对数据库无用的连接建立以及查询，占用数据库的资源**

**解决方案：布隆过滤器**

布隆过滤器的架构可以是

1.客户端自己实现bloom算法，自己承载bitmap数据（胖客户端）

2.客户端实现bloom算法，redis存放bitmap

3.redis实现bloom算法和bitmap

**布隆过滤器的缺点：1.只能增，不能删（删除时需要将value设置为null或error）；2.数据库增加了元素，需要对bloom也进行增加**

==针对布隆过滤器不能删除的缺点，可以使用布谷鸟过滤器或counting bloom==

## 雪崩

![803](3EE50320ADC44BDE99FCE31CB92C623B)

前提：redis作为缓存

雪崩有点像击穿，但是还是有区别的，

雪崩和击穿的区别：==击穿是某一个key有高并发请求访问；而雪崩是大量的key同时失效，间接造成大量的访问达到DB==

**发生的场景：某个时间点大量key过期**，例如：数据设置24小时有效期，零点数据清空

==解决方案：1.如果数据跟时点性无关，就设置随机过期时间==

==2.如果数据跟时点性有关，那么解决方案：1.业务层加判断，在零点延时==（零点延时：判断时间是零点时，数据访问随机睡几秒）+ ==强依赖击穿方案==（跟击穿的方案一样，访问请求加锁，setnx + 锁过期时间+ 多线程）【面试时一定要注意】

**注意：有些业务数据，就是要求零点数据清零，那么就是跟时点性有关**；

**如果零点时，数据访问量不大，可以只在业务层做零点延时**

做架构，就是要有取舍，要分析场景；另外呢，也不要把目光只放在redis上，要有全局观念

## 分布式锁

分布式锁就是：

1.setnx

2.设置锁过期时间

3.多线程(也是守护线程) 延长过期时间

而实现分布式锁最好的方案就是zookeeper

==面试时问到分布式锁要先将清楚上面3点，然后引出zookeeper==

## 数据预加载

大促的时候，如果已经准备好了大促的数据，可以往客户端做数据预加载，也就是为什么双十一大促的时候，提示先更新app，就是做数据预加载

如果大促时的数据没有准备好，则无法做预加载这个方案就行不通

# redis API  -初中级使用

## lettuce

https://github.com/lettuce-io/lettuce-core


```
同步方式
RedisClient client = RedisClient.create("redis://localhost");
StatefulRedisConnection<String, String> connection = client.connect();
RedisStringCommands sync = connection.sync();
String value = sync.get("key");
```


```
异步方式
StatefulRedisConnection<String, String> connection = client.connect();
RedisStringAsyncCommands<String, String> async = connection.async();
RedisFuture<String> set = async.set("key", "value")
RedisFuture<String> get = async.get("key")

async.awaitAll(set, get) == true

set.get() == "OK"
get.get() == "value"
```

==这个异步是客户端的异步，不是redis的异步，redis是不支持异步的==

大数据开发需要了解lettuce，架构方面不需要了解，直接看spring-redis就可以

就是简单的api：建立连接，取数据

## spring.data.redis

www.spring.io   ->  projects   -> spring data   ->  spring data redis

**注意spring data redis 是spring的集成方式，区别于Springboot中的redis集成**

**springboot redis只有redis的链接，只提供了SpringRedisTemplate对象**

    
```java

Spring Redis requires Redis 2.6 or above and Spring Data Redis integrates with Lettuce and Jedis, two popular open-source Java libraries for Redis.

#第一步：redis的连接
#RedisStandaloneConfiguration没有哨兵、没有集群
@Configuration
class RedisConfiguration {

  @Bean
  public JedisConnectionFactory redisConnectionFactory() {

    RedisStandaloneConfiguration config = new RedisStandaloneConfiguration("server", 6379);
    return new JedisConnectionFactory(config);
  }
}

#Redis Sentinel Support
/**
 * Jedis
 */
@Bean
public RedisConnectionFactory jedisConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new JedisConnectionFactory(sentinelConfig);
}

/**
 * Lettuce
 */
@Bean
public RedisConnectionFactory lettuceConnectionFactory() {
  RedisSentinelConfiguration sentinelConfig = new RedisSentinelConfiguration()
  .master("mymaster")
  .sentinel("127.0.0.1", 26379)
  .sentinel("127.0.0.1", 26380);
  return new LettuceConnectionFactory(sentinelConfig);
}

#Write to Master, Read from Replica
@Configuration
class WriteToMasterReadFromReplicaConfiguration {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
      .readFrom(SLAVE_PREFERRED)
      .build();

    RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration("server", 6379);

    return new LettuceConnectionFactory(serverConfig, clientConfig);
  }
}

#第二步：redis的使用
# 使用redis ： low-level/high level，高低阶API
high-level:  RedisTemplate，便于用户使用
low-level:RedisConnection，便于用户自定义使用

public class Example {

  // inject the actual template
  @Autowired
  private RedisTemplate<String, String> template;

  // inject the template as ListOperations
  @Resource(name="redisTemplate")
  private ListOperations<String, String> listOps;

  public void addLink(String userId, URL url) {
    listOps.leftPush(userId, url.toExternalForm());
  }
}


#第三步：经过什么样的编解码，将数据放入redis中
#Serializers
    
```


```
//springboot


```



```
redis项目

ce soft/
mkdir springboot
cd springboot
redis-server --port 6379

ipconfig  #查看redis所在机器的ip地址

redis-cli 
CONFIG GET *  #获取redis配置，安全策略protected-mode=yes,不开启保护模式，允许远端访问

CONFIG set protected-mode no  # 开启远端访问，只是临时更改


高阶API：RedisTemplate   StringRedisTemplate
//注意，存放到redis中的key,有乱码，也就是说高阶的RedisTemplate是面向java的序列化， 是要在key上加一些东西的，不是直接将key放到redis
//redis是二进制安全，存放 字节数组

//StringRedisTemplate   这时候存放到redis是期望的

低阶API：存放的是字节数组，存放在redis中的key、value是期望的


# hash

sean[age=22,name=zhouzhilei]

HGETALL sean 
HINCRBY sean age 1  #二进制安全，加1，尝试转为数组类型


# 对象Person

多系统，共享数据，要使用明文数据，例如json、xml

Hash mappers are converters of map objects to a Map<K, V> and back. HashMapper is intended for using with Redis Hashes.

Multiple implementations are available:
    
    BeanUtilsHashMapper using Spring’s BeanUtils.
    ObjectHashMapper using Object-to-Hash Mapping.
    Jackson2HashMapper using FasterXML Jackson.

Jackson2HashMapper
    Normal Mapping原生
    Flat Mapping扁平化

需要ObjectMapper,需要引入依赖：spring-boot-starter-json

在redis中存放的key是乱码，因为使用了RedisTemplate


# key/value自定义序列化

StringRedisSerializer

更好的序列化：Jackson2JsonRedisSerializer

这时候redis中的key、value就没有乱码了   

name-> "\"zhangsan"\"，双方在编解码时会剔除双引号，这个不是问题

HINCRBY sean01 age 1  //可以加1


放数据  -》   序列化 --》编解码


# 高阶API，指定序列化，便于使用 -> 自定义Template

自定义Template ->  MyTemplate

@Autowired   @Qualifier("ooxx)  指定容器中StringRedisTemplate的加载的Bean


# 扩展:发布订阅--初中阶使用

convertAndSend(channel,message)

redis-cli
SUBSCRIBE ooxx  #订阅数据

订阅是一直订阅，需要另外准备一个个线程，长期进行订阅
cc.subscribe()

PUBLISH ooxx hello  #别人发送消息，

自己发送消息

通过发布订阅，实现简易聊天室


```
