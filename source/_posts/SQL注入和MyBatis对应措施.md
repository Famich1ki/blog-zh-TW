---
title: SQL注入和MyBatis对应措施
date: 2024-08-28 16:07:19
tags:
  - SQL
  - MyBatis
categories:
  - [SQL, SQL注入]
cover: https://pics.findfuns.org/SQL-injection.jpg
---



## SQL注入

作为一个经典的网络安全问题，sql注入是每一个学习过网络安全课程的同学都多多少少接触过的一个问题。

它的原理其实非常简单，就是通过向sql语句中插入一些特殊的敏感字符，使得查询条件变成恒等从而绕过密码、用户名等的检查直接获取数据库的内容。更有甚者可以通过sql注入导致数据库内容损坏、丢失，产生一系列非常可怕的影响。

下面是一个sql注入的例子。

假如我有一个用于登录的表单，客户端可以输入用户名和密码来提交表单，后端通过对用户名和密码的验证返回数据。

```sql
SELECT * FROM users WHERE username = ${} AND password = ${};
```

此时，如果一些恶意用户试图输入一些特殊的符号来进行sql注入，如输入用户名为

```sql
' OR '1'='1
```

原先的sql就会变成一个恒等式

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
```

这样就完成了sql注入，用户将会得到数据库中全部的数据。

或者用户输入

```sql
admin'; DROP TABLE users; --
```

拼接之后就变成了

```sql
SELECT * FROM users WHERE username = 'admin'; DROP TABLE users; --'
```

在执行第一个sql之后，还会执行后面的DROP操作，导致users表直接被删除。

## 防护措施

1. 使用预编译的sql语句，避免输入直接和sql拼接，比如Java中的`PreparedStatement`
2. 使用ORM框架，如MyBatis、Hibernate等

下面介绍一下如何使用Mybatis框架来进行sql查询等操作。（当然也可以在SpringBoot中集成MyBatis，这里就不赘述了）。

首先在pom文件中导入相关依赖

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

编写mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <!-- 开启 SQL 日志 -->
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

日志的配置文件（log4j2，log4j，logback等等都可以）

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
      <!--想看到具体的sql需要调整日志级别到debug -->
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

mapper接口对应的xml文件（或者也可以直接在mapper的方法上使用注解，直接把sql写在注解上）

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

POJO类

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

给Mybatis编写一个Config类，在类中提供`sqlSessionFactory`的返回方法

```java
package com.zzb.config;

import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.apache.ibatis.io.Resources;

import java.io.InputStream;

public class MyBatisConfig {
    private static final SqlSessionFactory sqlSessionFactory;

    static { // 静态代码块，在类加载的时候就会运行，很适合建立数据库连接等操作
        try {
            String resource = "mybatis-config.xml"; // 加载mybatis配置文件
            InputStream inputStream = Resources.getResourceAsStream(resource); 
          	// 获取sqlSessionFactory
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

测试代码

```java
package com.zzb;

import com.zzb.config.MyBatisConfig;
import com.zzb.mapper.PersonMapper;
import com.zzb.pojo.User;
import org.apache.ibatis.session.SqlSession;

import java.util.List;

public class Main {
    public static void main(String[] args) {
      	// 使用try-with-resource获取sqlSession
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
User{id='zzb', name='zzb'}
```

在输出的日志中我们可以看到，实际上使用的是进行过预编译的sql语句，也就是说用户提供的参数不直接构成sql语句的结构，而只是作为参数值，这样就直接避免了sql注入。

MyBatis的预编译其实得益于JDBC底层的PreparedStatement，它会在执行sql查询前预先将sql语句进行预编译，在实际执行查询的时候再将预编译的sql填入参数。

如果这样说还不是很清楚，不妨看看下面这个例子。

这次我们不用`#{}`而是用`${}`。

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

此时输出的日志

```java
DEBUG [main] - ==>  Preparing: SELECT * FROM person_info WHERE name = 'zzb'
DEBUG [main] - ==> Parameters: 
DEBUG [main] - <==      Total: 1
User{id='zzb', name='zzb'}

```

不难看出，在使用`${}`时使用的sql是直接通过和参数拼接形成的，这种方式完全不能防止sql注入，风险很大！

而相比之下`#{}`安全得多，因为`#{}`使用的是预编译和参数绑定，在根本上防止了sql注入的发生。