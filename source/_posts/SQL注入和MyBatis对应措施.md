---
title: SQL注入和MyBatis對應措施
date: 2024-08-28 16:07:19
tags:
  - SQL
  - MyBatis
categories:
  - [SQL, SQL注入]
cover: https://pics.findfuns.org/SQL-injection.jpg
---



## SQL注入

作爲一個經典的網絡安全問題，sql注入是每一個學習過網絡安全課程的同學都多多少少接觸過的一個問題。

它的原理其實非常簡單，就是通過向sql語句中插入一些特殊的敏感字符，使得查詢條件變成恆等從而繞過密碼、用戶名等的檢查直接獲取數據庫的內容。更有甚者可以通過sql注入導緻數據庫內容損壞、丟失，産生一繫列非常可怕的影響。

下麵是一個sql注入的例子。

假如我有一個用於登錄的表單，客戶端可以輸入用戶名和密碼來提交表單，後端通過對用戶名和密碼的驗証返回數據。

```sql
SELECT * FROM users WHERE username = ${} AND password = ${};
```

此時，如果一些噁意用戶試圖輸入一些特殊的符號來進行sql注入，如輸入用戶名爲

```sql
'' OR ''1''=''1
```

原先的sql就會變成一個恆等式

```sql
SELECT * FROM users WHERE username = '''' OR ''1''=''1'' AND password = '''';
```

這樣就完成了sql注入，用戶將會得到數據庫中全部的數據。

或者用戶輸入

```sql
admin''; DROP TABLE users; --
```

拼接之後就變成了

```sql
SELECT * FROM users WHERE username = ''admin''; DROP TABLE users; --''
```

在執行第一個sql之後，還會執行後麵的DROP操作，導緻users表直接被刪除。

## 防護措施

1. 使用預編譯的sql語句，避免輸入直接和sql拼接，比如Java中的`PreparedStatement`
2. 使用ORM框架，如MyBatis、Hibernate等

下麵介紹一下如何使用Mybatis框架來進行sql查詢等操作。（當然也可以在SpringBoot中集成MyBatis，這裡就不贅述了）。

首先在pom文件中導入相關依賴

```xml
<!-- https://mvnrepository.com/artifact/com.mysql/mysql-connector-j -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.0.33</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.6</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-core -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.20.0</version>
</dependency>
```

編冩mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <!-- 開啟 SQL 日誌 -->
        <setting name="logImpl" value="LOG4J2"/>
    </settings>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/person"/>
                <property name="username" value="root"/>
                <property name="password" value="123"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="com/zzb/mapper/PersonMapper.xml"/>
    </mappers>


</configuration>
```

日誌的配置文件（log4j2，log4j，logback等等都可以）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration xmlns="http://logging.apache.org/log4j/2.0/config">

    <Appenders>
        <Console name="stdout" target="SYSTEM_OUT">
            <PatternLayout pattern="%5level [%t] - %msg%n"/>
        </Console>
    </Appenders>

    <Loggers>
        <Logger name="com.zzb.mapper.PersonMapper" level="debug"/> 
      <!--想看到具體的sql需要調整日誌級別到debug -->
        <Root level="error" >
            <AppenderRef ref="stdout"/>
        </Root>
    </Loggers>

</Configuration>
```

mapper接口

```java
package com.zzb.mapper;

public interface PersonMapper {
    public List<User> selectUser(@Param(value="name") String name);
}
```

mapper接口對應的xml文件（或者也可以直接在mapper的方法上使用注解，直接把sql冩在注解上）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zzb.mapper.PersonMapper">

    <select id="selectUser" resultType="com.zzb.pojo.User">
        SELECT * FROM person_info WHERE name = #{name}
    </select>

</mapper>
```

POJO類

```java
package com.zzb.pojo;

public class User {
    private String id;
    private String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    public User() {
    }
  
  	// getter, setter and toString
}
```

給Mybatis編冩一個Config類，在類中提供`sqlSessionFactory`的返回方法

```java
package com.zzb.config;

import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.apache.ibatis.io.Resources;

import java.io.InputStream;

public class MyBatisConfig {
    private static final SqlSessionFactory sqlSessionFactory;

    static { // 靜態代碼塊，在類加載的時候就會運行，很適合建立數據庫連接等操作
        try {
            String resource = "mybatis-config.xml"; // 加載mybatis配置文件
            InputStream inputStream = Resources.getResourceAsStream(resource); 
          	// 獲取sqlSessionFactory
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
	
    public static SqlSessionFactory getSqlSessionFactory() {
        return sqlSessionFactory;
    }
}
```

測試代碼

```java
package com.zzb;

import com.zzb.config.MyBatisConfig;
import com.zzb.mapper.PersonMapper;
import com.zzb.pojo.User;
import org.apache.ibatis.session.SqlSession;

import java.util.List;

public class Main {
    public static void main(String[] args) {
      	// 使用try-with-resource獲取sqlSession
        try (SqlSession session = MyBatisConfig.getSqlSessionFactory().openSession()) {
            PersonMapper mapper = session.getMapper(PersonMapper.class);
            List<User> user = mapper.selectUser("zzb");
            for(User usr: user) {
                System.out.println(usr);
            }
        }
    }
}
```

OutPut

```java
DEBUG [main] - ==>  Preparing: SELECT * FROM person_info WHERE name = ?
DEBUG [main] - ==> Parameters: zzb(String)
DEBUG [main] - <==      Total: 1
User{id=''zzb'', name=''zzb''}
```

在輸出的日誌中我們可以看到，實際上使用的是進行過預編譯的sql語句，也就是説用戶提供的參數不直接構成sql語句的結構，而隻是作爲參數值，這樣就直接避免了sql注入。

MyBatis的預編譯其實得益於JDBC底層的PreparedStatement，它會在執行sql查詢前預先將sql語句進行預編譯，在實際執行查詢的時候再將預編譯的sql填入參數。

如果這樣説還不是很清楚，不妨看看下麵這個例子。

這次我們不用`#{}`而是用`${}`。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zzb.mapper.PersonMapper">

    <select id="selectUser" resultType="com.zzb.pojo.User">
        SELECT * FROM person_info WHERE name = ${name}
    </select>

</mapper>
```

此時輸出的日誌

```java
DEBUG [main] - ==>  Preparing: SELECT * FROM person_info WHERE name = ''zzb''
DEBUG [main] - ==> Parameters: 
DEBUG [main] - <==      Total: 1
User{id=''zzb'', name=''zzb''}

```

不難看出，在使用`${}`時使用的sql是直接通過和參數拼接形成的，這種方式完全不能防止sql注入，風險很大！

而相比之下`#{}`安全得多，因爲`#{}`使用的是預編譯和參數綁定，在根本上防止了sql注入的髮生。