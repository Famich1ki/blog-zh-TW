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

説起Nosql數據庫，或者數據庫緩存，消息隊列，token認証等等應用時，Redis總是大家繞不開的話題。作爲一款由C語言開髮的運行在內存中進行數據讀冩的應用，它以高性能和易用性著稱。得益於單線程的設計，避免了多線程場景下的鎖競爭，從而能做到每秒數十萬次的讀冩操作。同時，Redis還實現了主從模式，哨兵模式和集群模式等分佈式解決方案，在保証高性能的前提下提高了可用性和擴展性，成爲了開髮時不可或缺的組件。

## Spring Boot 整合 Redis

由於Spring Boot中各種starter的存在，配置redis其實是簡單的事情。

**首先**在pom中導入依賴

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

`spring-boot-starter-data-redis`是redis的spring boot依賴，其中包含了`lettuce`客戶端。redis的客戶端可以選擇`lettuce`或`jedis`，前者是默認的，而且支持異步，在並髮場景下的性能更好，推薦使用。

`jackson-databind`這個是jackson的依賴，用於之後配置redis的序列化。

`commons-pool2`用於配置redis集群。

**接下來**是redis的config

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
      	// 創建一個RedisTemplate實例，它封裝了操作數據的各種方法。
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        // 使用factory來配置redisTemplate，負責底層的數據庫連接工作。
        redisTemplate.setConnectionFactory(lettuceConnectionFactory);
			  // 自定義序列化
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

對於Redis自定義序列化，首先要知道爲什麼需要自定義配置，難道不主動去配置序列化就不能使用了嗎？

其實不是，redis已經提供給了我們默認的序列化方法，翻閱[官方文檔](https://docs.spring.io/spring-data/redis/reference/redis/template.html)就可以看到如下內容

![](https://pics.findfuns.org/redis-default-serializer.png)



文中説的意思是`JdkSerializationRedisSerializer`是`RedisTemplate`和`RedisCache`默認的序列化方法。這個默認的序列化方法也就是JDK自帶的序列化方法（實現了Serializable接口）。確實如此，如果不進行自定義配置，Redis也是可以用默認的設置進行序列化的。

但是默認方法有一個缺點，當我們將數據存入redis之後，在redis中存儲的實際上是序列化之後的格式，也就是無法直接閱讀的格式，不利於檢查和調試，下麵是一個例子。

當我們用默認的序列化向redis中添加鍵值對("k1", "v1")，然後啟動redis-cli查看key的是

![](https://pics.findfuns.org/redis-default-k1.png)

在redis中存儲的是序列化後的形式，前麵的一堆16進製是JDK在序列化時添加的一些前綴信息，如果想獲得存入時原始的格式，隻能通過`redisTemplate.opsForValue().get("k1")`來獲取。

下麵來介紹一下自定義的序列化方法

- `GenericJackson2JsonRedisSerializer`
- `Jackson2JsonRedisSerializer`

這兩種序列化方式在序列化String，Object，Collections，JSONObject，JSONArray都完全沒有問題，唯一的區別是，當使用`GenericJackson2JsonRedisSerializer`時，每個對象上會加上一個`@class`屬性，屬性值就是對應的全類名，而後者沒有這個`@class`屬性，所以在使用`Jackson2JsonRedisSerializer`時如果強製類型轉換會報錯，所以推薦使用`GenericJackson2JsonRedisSerializer`。

此外，需要用一個objectMapper來配置`GenericJackson2JsonRedisSerializer`，否則在序列化JSONObject的時候會出錯

```java
objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
```

`setVisibility`方法可以設定訪問類的字段和方法的權限

<center>
  <img src="https://pics.findfuns.org/propertyAccessor.png" style="zoom:40%;">
  <img src="https://pics.findfuns.org/visibility.png" style="zoom:40%;" />
</center>

```java
objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
```

`activateDefaultTyping`這個方法用於配置處理多態時能夠正確的序列化和反序列化。

`LaissezFaireSubTypeValidator`是一個比較寬鬆的類型驗証器，instance是一個靜態成員，實際上就是返回了一個`LaissezFaireSubTypeValidator`的實例

```java
public static final LaissezFaireSubTypeValidator instance = new LaissezFaireSubTypeValidator();
```

`DefaultTyping`指定如何處理類型信息

<img src="https://pics.findfuns.org/defaultTyping.png" style="zoom:33%;" />

#### 封裝一個RedisUtils類

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

包含了簡單的set和get方法以及獲取所有的key。

#### 測試類

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

在redis-cli中嚐試獲取剛才的數據

<img src="https://pics.findfuns.org/redis-console.png" style="zoom:75%;" />

美化一下JSON之後得到的輸出(字符串v1就不展示了)

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

