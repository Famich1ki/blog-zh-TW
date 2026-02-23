---
title: Spring Boot整合Redis和Redis配置
date: 2024-09-02 17:01:21
tags: 
  - Spring Boot 
  - Redis
  - JSON
categories:
  - [Spring Boot, Redis]
cover: https://pics.findfuns.org/springboot-redis.png
---
## 前言

说起Nosql数据库，或者数据库缓存，消息队列，token认证等等应用时，Redis总是大家绕不开的话题。作为一款由C语言开发的运行在内存中进行数据读写的应用，它以高性能和易用性著称。得益于单线程的设计，避免了多线程场景下的锁竞争，从而能做到每秒数十万次的读写操作。同时，Redis还实现了主从模式，哨兵模式和集群模式等分布式解决方案，在保证高性能的前提下提高了可用性和扩展性，成为了开发时不可或缺的组件。

## Spring Boot 整合 Redis

由于Spring Boot中各种starter的存在，配置redis其实是简单的事情。

**首先**在pom中导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

`spring-boot-starter-data-redis`是redis的spring boot依赖，其中包含了`lettuce`客户端。redis的客户端可以选择`lettuce`或`jedis`，前者是默认的，而且支持异步，在并发场景下的性能更好，推荐使用。

`jackson-databind`这个是jackson的依赖，用于之后配置redis的序列化。

`commons-pool2`用于配置redis集群。

**接下来**是redis的config

```java
@Configuration
public class RedisConfig {

    // Jackson2JsonRedisSerializer
    // GenericJackson2JsonRedisSerializer
    // RedisSerializer
    // ObjectMapper
    // LaissezFaireSubTypeValidator
    @Bean(name = "myRedisTemplate")
    public RedisTemplate<String, Object> getRedisTemplate(LettuceConnectionFactory lettuceConnectionFactory) {
      	// 创建一个RedisTemplate实例，它封装了操作数据的各种方法。
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        // 使用factory来配置redisTemplate，负责底层的数据库连接工作。
        redisTemplate.setConnectionFactory(lettuceConnectionFactory);
			  // 自定义序列化
       	StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer = getSerializer();

        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(genericJackson2JsonRedisSerializer);
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(genericJackson2JsonRedisSerializer);

        return redisTemplate;
    }

    public GenericJackson2JsonRedisSerializer getSerializer() {
      	ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, 	ObjectMapper.DefaultTyping.NON_FINAL);
        return new GenericJackson2JsonRedisSerializer(objectMapper);
    }
}
```

#### 序列化配置

对于Redis自定义序列化，首先要知道为什么需要自定义配置，难道不主动去配置序列化就不能使用了吗？

其实不是，redis已经提供给了我们默认的序列化方法，翻阅[官方文档](https://docs.spring.io/spring-data/redis/reference/redis/template.html)就可以看到如下内容

![](https://pics.findfuns.org/redis-default-serializer.png)



文中说的意思是`JdkSerializationRedisSerializer`是`RedisTemplate`和`RedisCache`默认的序列化方法。这个默认的序列化方法也就是JDK自带的序列化方法（实现了Serializable接口）。确实如此，如果不进行自定义配置，Redis也是可以用默认的设置进行序列化的。

但是默认方法有一个缺点，当我们将数据存入redis之后，在redis中存储的实际上是序列化之后的格式，也就是无法直接阅读的格式，不利于检查和调试，下面是一个例子。

当我们用默认的序列化向redis中添加键值对("k1", "v1")，然后启动redis-cli查看key的是

![](https://pics.findfuns.org/redis-default-k1.png)

在redis中存储的是序列化后的形式，前面的一堆16进制是JDK在序列化时添加的一些前缀信息，如果想获得存入时原始的格式，只能通过`redisTemplate.opsForValue().get("k1")`来获取。

下面来介绍一下自定义的序列化方法

- `GenericJackson2JsonRedisSerializer`
- `Jackson2JsonRedisSerializer`

这两种序列化方式在序列化String，Object，Collections，JSONObject，JSONArray都完全没有问题，唯一的区别是，当使用`GenericJackson2JsonRedisSerializer`时，每个对象上会加上一个`@class`属性，属性值就是对应的全类名，而后者没有这个`@class`属性，所以在使用`Jackson2JsonRedisSerializer`时如果强制类型转换会报错，所以推荐使用`GenericJackson2JsonRedisSerializer`。

此外，需要用一个objectMapper来配置`GenericJackson2JsonRedisSerializer`，否则在序列化JSONObject的时候会出错

```java
objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
```

`setVisibility`方法可以设定访问类的字段和方法的权限

<center>
  <img src="https://pics.findfuns.org/propertyAccessor.png" style="zoom:40%;">
  <img src="https://pics.findfuns.org/visibility.png" style="zoom:40%;" />
</center>

```java
objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
```

`activateDefaultTyping`这个方法用于配置处理多态时能够正确的序列化和反序列化。

`LaissezFaireSubTypeValidator`是一个比较宽松的类型验证器，instance是一个静态成员，实际上就是返回了一个`LaissezFaireSubTypeValidator`的实例

```java
public static final LaissezFaireSubTypeValidator instance = new LaissezFaireSubTypeValidator();
```

`DefaultTyping`指定如何处理类型信息

<img src="https://pics.findfuns.org/defaultTyping.png" style="zoom:33%;" />

#### 封装一个RedisUtils类

```java
package org.zzb.redisusage.utils;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Set;

@Component
public class RedisUtils {

    private final RedisTemplate<String, Object> redisTemplate;

    @Autowired
    public RedisUtils(@Qualifier("myRedisTemplate") RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    public Set<String> getAllKeys() {
        return redisTemplate.keys("*");
    }
}
```

包含了简单的set和get方法以及获取所有的key。

#### 测试类

```java
@SpringBootTest
@Slf4j
class RedisUsageApplicationTests {

    @Autowired
    RedisUtils redisUtils;

    @Test
    void contextLoads() throws JSONException {
        // Java Pojo
        String JavaPojo = "JavaPojo";
        Person person01 = new Person("zzb", "English");
        redisUtils.set(JavaPojo, person01);

        // JSONObject
        String JSONObjectKey = "JSONObjectKey";
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("name", "zzb");
        jsonObject.put("age", 23);
        redisUtils.set(JSONObjectKey, jsonObject);

        // String
        String JavaString = "JavaString";
        redisUtils.set(JavaString, "v1");

        // Collections -> List
        String JavaList = "JavaList";
        List<Person> list = new ArrayList<>();
        list.add(person01);
        redisUtils.set(JavaList, list);

        // JSONArray
        String JSONArray = "JSONArray";
        JSONArray jsonArray = new JSONArray();
        jsonArray.put(person01);
        redisUtils.set(JSONArray, jsonArray);

        log.info("JavaPojo -> {}", redisUtils.get(JavaPojo));
        log.info("JavaString -> {}", redisUtils.get(JavaString));
        log.info("JSONObject -> {}", redisUtils.get(JSONObjectKey));
        log.info("ArrayList -> {}", redisUtils.get(JavaList));
        log.info("JSONArray -> {}", redisUtils.get(JSONArray));

        log.info("All keys -> {}", redisUtils.getAllKeys());
    }
}
```

Output

```java
2024-09-02T16:46:24.048+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : JavaPojo -> Person(name=zzb, language=English)
2024-09-02T16:46:24.050+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : JavaString -> v1
2024-09-02T16:46:24.056+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : JSONObject -> {"name":"zzb","age":23}
2024-09-02T16:46:24.059+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : ArrayList -> [Person(name=zzb, language=English)]
2024-09-02T16:46:24.061+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : JSONArray -> ["Person(name=zzb, language=English)"]
2024-09-02T16:46:24.066+08:00  INFO 61606 --- [redis-usage] [           main] o.z.r.RedisUsageApplicationTests         : All keys -> [JSONObjectKey, JavaString, JavaPojo, JSONArray, JavaList]
```

在redis-cli中尝试获取刚才的数据

<img src="https://pics.findfuns.org/redis-console.png" style="zoom:75%;" />

美化一下JSON之后得到的输出(字符串v1就不展示了)

```Json
[
  "org.zzb.redisusage.pojo.Person",
  {
    "name": "zzb",
    "language": "English"
  }
]

[
  "org.json.JSONObject",
  {
    "nameValuePairs": [
      "java.util.HashMap",
      {
        "name": "zzb",
        "age": 23
      }
    ]
  }
]

[
  "java.util.ArrayList",
  [
    [
      "org.zzb.redisusage.pojo.Person",
      {
        "name": "zzb",
        "language": "English"
      }
    ]
  ]
]

[
  "org.json.JSONArray",
  {
    "values": [
      "java.util.ArrayList",
      [
        [
          "org.zzb.redisusage.pojo.Person",
          {
            "name": "zzb",
            "language": "English"
          }
        ]
      ]
    ]
  }
]
```

