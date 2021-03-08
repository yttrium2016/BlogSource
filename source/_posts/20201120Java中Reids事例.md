---
title: Java中Reids事例
tags:
  - linux
  - redis
  - java
  - 分布式
categories: 后台
abbrlink: 4969
date: 2020-11-20 20:00:00
---

## Java中Reids事例

### Jedis 调用

#### 引入maven依赖

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

#### Main.java 代码调用

```
public static void main(String[] args) {
        //连接本地的 Redis 服务
        Jedis jedis = new Jedis("localhost");
        System.out.println("连接成功");
 
        // 获取数据并输出
        Set<String> keys = jedis.keys("*"); 
        Iterator<String> it=keys.iterator() ;   
        while(it.hasNext()){   
            String key = it.next();   
            System.out.println(key);   
        }
}
```

> jedis 所有的API和redis的API一模一样 同理

#### hutool 工具调用 

[官网](https://www.hutool.cn/)  [Redis工具类调用教程](https://www.hutool.cn/docs/#/db/NoSQL/Redis%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%B0%81%E8%A3%85-RedisDS?id=%e6%9e%84%e5%bb%ba)

```java
Jedis jedis = RedisDS.create().getJedis();
```

### SpringBoot 调用

#### 引入maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

#### 配置地址

```yaml
spring:
  redis:
    port: 6379
    host: 10.10.10.228
    database: 0
    password:
    timeout: 6000ms
```

#### 配置RedisConfig.java

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory){
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(factory);
        StringRedisSerializer serializer = new StringRedisSerializer();
        redisTemplate.setKeySerializer(serializer);
        redisTemplate.setValueSerializer(serializer);
        redisTemplate.setHashKeySerializer(serializer);
        redisTemplate.setHashValueSerializer(serializer);
        return redisTemplate;
    }

    @Bean
    public ValueOperations<String,String> valueOperations(RedisTemplate<String, String> redisTemplate){
        return redisTemplate.opsForValue();
    }

    @Bean
    public HashOperations<String,String,Object> hashOperations(RedisTemplate<String, Object> redisTemplate){
        return redisTemplate.opsForHash();
    }

    @Bean
    public ListOperations<String,Object> listOperations(RedisTemplate<String, Object> redisTemplate){
        return redisTemplate.opsForList();
    }

    @Bean
    public SetOperations<String,Object> setOperations(RedisTemplate<String, Object> redisTemplate){
        return redisTemplate.opsForSet();
    }

    @Bean
    public ZSetOperations<String,Object> zSetOperations(RedisTemplate<String, Object> redisTemplate){
        return redisTemplate.opsForZSet();
    }
}

```

#### 使用

```java
@Autowired
private RedisTemplate<String, Object> redisTemplate;
```

### 自己实现的Redis分布式锁

#### 简易实现

```java
@Controller
public class TestController {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @RequestMapping("/buy")
    @ResponseBody
    public Object buy() {

        String key = "redisLuck";
        String value = UUID.randomUUID().toString();

        Boolean flag = redisTemplate.opsForValue().setIfAbsent(key, value, 30, TimeUnit.SECONDS);
        if (flag == null || !flag) {
            return "抢购失败,请稍后再试.";
        }

        try {
            orderService.buy();
            return "抢购成功";
        } finally {
            if (Objects.equals(redisTemplate.opsForValue().get(key), value)) {
                redisTemplate.delete(key);
            }
        }

    }
}
```

> 利用 SETNX 当锁,通过同一个Key去执行 通过设置超时防止死锁,通过UUID的value来防止多个抢购解除非自己加的锁
>
> 问题: 如果逻辑操作非常复杂 超过30S的话还是会出现(超卖)问题.因为redis是单线程的 没有非常非常高的并发性能

#### 优化 (Redisson)

思路: 加锁的时候开另一个线程去检测 程序是否正常在运行,在的话 延迟锁的时间.

方法: 使用Redisson库

Redisson是架设在[Redis](https://baike.baidu.com/item/Redis/6549233)基础上的一个[Java](https://baike.baidu.com/item/Java/85979)驻内存数据[网格](https://baike.baidu.com/item/网格/265734)（In-Memory Data Grid）。【Redis官方推荐】

Redisson在基于NIO的[Netty](https://baike.baidu.com/item/Netty/10061624)框架上，充分的利用了Redis键值数据库提供的一系列优势，在Java实用工具包中常用接口的基础上，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多[线程](https://baike.baidu.com/item/线程/103101)并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。

#### 例子优化

##### 引入maven依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.15.0</version>
</dependency>
```

##### 配置Config.java (SpringBoot的配置)

```java
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private String port;
    
    @Bean
    public RedissonClient redissonClient(){
        Config config = new Config();
        String address = "redis://"+host+":"+port;
        config.useSingleServer().setAddress(address);
        return Redisson.create(config);
    }
}
```

##### 抢购优化代码

```java
@Controller
public class TestController {

    @Autowired
    private RedissonClient redissonClient;

    @RequestMapping("/buy")
    @ResponseBody
    public Object buy() {

        String key = "redisLuck";
	    RLock lock = redissonClient.getLock(key);

        try {
            lock.lock(30,TimeUnit.SECONDS);
            return "抢购成功";
        } finally {
            lock.unlock();
        }

    }
}
```

> lock.lock(30,TimeUnit.SECONDS); 会开个子线程 每10秒(默认设置时间的1/3)去检测是否正常运行 重新把超时时间改为30秒过期
>
> 注意:如果一次性很高的并发量请求过来的时候,后续的请求会被阻塞容易造成大量请求的堆积

#### 注解优化

通过spring aop 改造

##### 注解 RedisLuck

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RedisLuck {
    String name() default "redis_luck";
}
```

##### 增加代码 RedisLuckAop.java

```java
@Aspect
@Component
public class RedisLuckAop {

    @Autowired
    private RedissonClient redissonClient;

    @Around("@annotation(cn.com.yangzhenyu.redisboot.annotation.RedisLuck)")
    public Object aopRedisson(ProceedingJoinPoint point) throws Throwable{
        Object result = null;

        MethodSignature signature = (MethodSignature) point.getSignature();
        RedisLuck redisLuck = signature.getMethod().getAnnotation(RedisLuck.class);
        if (null == redisLuck) {
            return point.proceed();
        }

        RLock lock = redissonClient.getLock(redisLuck.value());

        try {
            lock.lock(30,TimeUnit.SECONDS);
            result = point.proceed();
        } finally {
            lock.unlock();
        }
        return result;
    }
}
```

##### 使用

在需要使用的加锁的方法上加上注解 @RedisLuck("...")

```
@Override
@RedisLuck("luck")
public void buy() {
	// ....
}
```

例子: [https://github.com/yttrium2016/reids-boot](https://github.com/yttrium2016/reids-boot)

### Redisson 实现布隆过滤器

解决缓存穿透 

> 缺点 : 但是布隆过滤器的缺点和优点一样明显。误算率是其中之一。随着存入的元素数量增加，误算率随之增加。常见的补救办法是建立一个小的白名单，存储那些可能被误判的元素。但是如果元素数量太少，则使用散列表足矣。
>
> 另外，一般情况下不能从布隆过滤器中删除元素。我们很容易想到把位列阵变成整数数组，每插入一个元素相应的计数器加1, 这样删除元素时将计数器减掉就可以了。然而要保证安全的删除元素并非如此简单。首先我们必须保证删除的元素的确在布隆过滤器里面. 这一点单凭这个过滤器是无法保证的。另外计数器回绕也会造成问题。
>
> 在降低误算率方面，有不少工作，使得出现了很多布隆过滤器的变种。

#### 代码实现

```java
//初始化
RBloomFilter<Object> filter = redissonClient.getBloomFilter("blFilter");
// length 数据大致长度 v 误差率(越低,存储消耗越大)
filter.tryInit(length, v);
//添加元素
filter.add(xx);
//检测是否存储
filter.contains(xx); //true | false
```

> [布隆过滤器原理](https://www.cnblogs.com/cpselvis/p/6265825.html)

### SpringBoot整合Reids做Session共享

集群环境中,在session保存登录人的信息的时候,如果2次请求在2台服务器上,拿不到前一次的登录信息,所以引入Redis做全局的统一的session保存登录信息

#### 创建一个基础的SpringBoot项目

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.com.yangzhenyu</groupId>
    <artifactId>redis-session</artifactId>
    <version>1.0.0</version>
    <name>redis-session</name>
    <description>Redis Session project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.5.8</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

##### RedisController.java

```java
@RestController
public class RedisController {

    @Value("${server.port}")
    private String port;

    @Autowired
    private HttpSession httpSession;

    // 设置Session
    @RequestMapping("set")
    public String setSession(@RequestParam(required = false) String key, String value) {
        HostInfo hostInfo = SystemUtil.getHostInfo();
        if (StrUtil.isBlank(key)) {
            key = "key";
        }
        httpSession.setAttribute(key, value);
        return port + ":" + "ok";
    }

    // 读取Session
    @RequestMapping("get")
    public String getSession(String key) {
        HostInfo hostInfo = SystemUtil.getHostInfo();
        if (StrUtil.isBlank(key)) {
            key = "key";
        }
        return port + ":" + httpSession.getAttribute(key);
    }
}
```

##### application.properties

```properties
spring.redis.host=127.0.0.1
spring.redis.port=6379
```

##### RedisSessionConfig.java

```java
@Configuration
// 开启RedisHttpSession
@EnableRedisHttpSession
public class RedisSessionConfig {
}
```

> 已经完成了多个服务器共享Redis Session了

#### 测试

##### 启动

![image-20210305145133565](http://img.yzy.ink/image-20210305145133565.png)

> 启动2个不同端口的项目

使用nginx简单的负载均衡

```
upstream appserver {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
server {
    listen 9000;
    location / {
    	proxy_pass http://appserver;
    }
}
```

> 注意:[appserver] 不能有[_]下划线 不然TOMCAT高版本会报错

![image-20210305145446021](http://img.yzy.ink/image-20210305145446021.png)

##### 请求

###### 设置值

![image-20210305145553658](http://img.yzy.ink/image-20210305145553658.png)

###### 测试请求

![image-20210305145633190](http://img.yzy.ink/image-20210305145633190.png)

![image-20210305145652905](http://img.yzy.ink/image-20210305145652905.png)

> 2台机器访问都能拿到统一的数据