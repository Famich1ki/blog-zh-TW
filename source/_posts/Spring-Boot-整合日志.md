---
title: Spring Boot 整合日志
date: 2024-08-30 19:05:59
tags:
  - Spring Boot
  - 日志
categories:
  - [Spring Boot, 日志]
cover: https://pics.findfuns.org/springboot.png
---
## 前言

Java发展至今已经形成了一套完整的日志体系，分为日志门面和日志具体实现。日志门面相当于一个接口，日志各种实现其实就是日志门面的不同实现。

Java的日志加载依赖于**SPI**，不同于API， SPI模式下接口处于调用者一侧，开发者一侧只拥有具体的实现。而API模式下接口和具体实现都处于开发者一侧，调用者无需关心具体实现，只管调用开发者提供的接口即可。

SPI机制给予了程序动态加载的能力，程序运行的过程中根据接口去动态扫描对应的接口实现类并加载。

举个例子，Java的日志门面Slf4j对应的具体实现有Spring Boot自带的也是默认的logback，还有log4j，log4j2等，在使用时会动态加载classpath中存在的slf4j对应的实现类并把它加载使用，这就是SPI机制。

<img src="https://pics.findfuns.org/java-log-structure.png" style="zoom:50%;" />

### Logback

logback是Spring Boot默认使用的日志实现，即使不配置也可以直接使用。

```java
@SpringBootApplication
public class RedisUsageApplication {

    public static void main(String[] args) {
        SpringApplication.run(RedisUsageApplication.class, args);
      	Logger log = LoggerFactory.getLogger("RedisUsageApplication.class");
	log.warn("Easy bro, this is a fake warn lol.");
        log.error("Easy bro, this is a fake error lol.");
    }
}
```

也可以使用Lombok提供的注解`@Slf4j`来输出log，这样简洁一些。

```java
@SpringBootApplication
@Slf4j
public class RedisUsageApplication {

    public static void main(String[] args) {
	SpringApplication.run(RedisUsageApplication.class, args);
	log.warn("Easy bro, this is a fake warn lol.");
        log.error("Easy bro, this is a fake error lol.");
    }
}
```



<img src="https://pics.findfuns.org/logback-default.png" style="zoom:200%;" />

但是默认的配置还是不能够更加精准的满足各种需求，需要进行自定义设置。

在classpath下创建logback.xml或者logback-spring.xml

```xml
<!--
logback的日志级别： ERROR > WARN > INFO > DEBUG > TRACE
-->
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <!-- 定义日志文件名称 -->
    <property name="APP_NAME" value="log-files" />
    <!-- 定义日志文件的路径 -->
    <property name="LOG_PATH" value="${user.home}/${APP_NAME}" />
    <!-- 定义日志的文件名 -->
    <property name="LOG_FILE" value="${LOG_PATH}/redis-usage.log" />

    <!-- 滚动记录日志，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件 -->
    <appender name="APPLICATION"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 指定日志文件的名称 -->
        <file>${LOG_FILE}</file>
        <!--
          当发生滚动时，决定 RollingFileAppender 的行为，涉及文件移动和重命名
          TimeBasedRollingPolicy： 最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责触发滚动。
          -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--
           滚动时产生的文件的存放位置及文件名称
           %d{yyyy-MM-dd}：按天进行日志滚动
           %i：当文件大小超过maxFileSize时，按照i进行文件滚动
           -->
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!--
           可选节点，控制保留的归档文件的最大数量，超出数量就删除旧文件。假设设置每天滚动，
           且maxHistory是7，则只保存最近7天的文件，删除之前的旧文件。
           注意，删除旧文件时，那些为了归档而创建的目录也会被删除。
           -->
            <maxHistory>7</maxHistory>
            <!--
           当日志文件超过maxFileSize指定的大小时，根据上面提到的%i进行日志文件滚动
           注意此处配置SizeBasedTriggeringPolicy是无法实现按文件大小进行滚动的，
           必须配置timeBasedFileNamingAndTriggeringPolicy
           -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>50MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <!-- 日志输出格式： -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [ %thread ] - [ %-5level ] [ %logger{50} : %line ] - %msg%n</pattern>
        </layout>
    </appender>
    <!-- ch.qos.logback.core.ConsoleAppender 表示控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--
       日志输出格式：
           %d表示日期时间，%green 绿色
           %thread表示线程名，%magenta 洋红色
           %-5level：级别从左显示5个字符宽度 %highlight 高亮色
           %logger{36} 表示logger名字最长36个字符，否则按照句点分割 %yellow 黄色
           %msg：日志消息
           %n是换行符
       -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%green(%d{yyyy-MM-dd HH:mm:ss.SSS}) [%magenta(%thread)] %highlight(%-5level) %yellow(%logger{36}): %msg%n</pattern>
        </layout>
    </appender>

    <!--
   root与logger是父子关系，没有特别定义则默认为root，任何一个类只会和一个logger对应，
   要么是定义的logger，要么是root，判断的关键在于找到这个logger，然后判断这个logger的appender和level。
   -->
    <root level="info">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="APPLICATION" />
    </root>
</configuration>
```

log的配置中有这么几个标签

- property： 可以自定义变量名和变量值，方便拼接路径等。
- appender： 每一个appender对应一个日志的输出地点
  - layout： 日志的输出格式
  - rollingPolicy： 日志的滚动策略

通过这些标签可以实现日志的完全自定义。

使用logback时可以在application.yml或者application.properties中加入logback的配置，但也可以不配置，因为Spring Boot的默认行为是扫描classpath下是否有logback.xml或者logback-spring.xml。

```yaml
logging:
  config: classpath:logback-spring.xml
```

![](https://pics.findfuns.org/logback-customized.png)

同时日志文件也可以在对应的目录下找到

<img src="https://pics.findfuns.org/log-files.png" style="zoom:50%;" />

## Log4j2

log4j2和logback都是slf4j的具体实现，但在实际应用中二者不能同时存在，否则会报错，导致无法正常加载日志的实现类。

所以在使用log4j2时需要在pom中显示的剔除logback依赖，同时加入log4j2的依赖。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <!--剔除logback的依赖-->
  	<exclusions>
      <exclusion>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
      </exclusion>
  	</exclusions>
</dependency>
<!--log4j2依赖-->
	<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

在classpath下创建log4j2.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- log4j2的日志级别： OFF > FATAL > ERROR > WARN > INFO > DEBUG > ALL -->
<Configuration>
    <!--<Configuration status="WARN" monitorInterval="30"> -->
    <properties>
        <property name="LOG_HOME">./service-logs</property>
    </properties>
    <Appenders>
        <!--*********************控制台日志***********************-->
        <Console name="consoleAppender" target="SYSTEM_OUT">
            <!--设置日志格式及颜色-->
            <PatternLayout
                    pattern="%style{%d{ISO8601}}{bright,green} %highlight{%-5level} [%style{%t}{bright,blue}] %style{%C{}}{bright,yellow}: %msg%n%style{%throwable}{red}"
                    disableAnsi="false" noConsoleNoAnsi="false"/>
        </Console>

        <!--*********************文件日志***********************-->
        <!--all级别日志-->
        <RollingFile name="allFileAppender"
                     fileName="${LOG_HOME}/all.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/all-%d{yyyy-MM-dd}-%i.log.gz">
            <!--设置日志格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 设置日志文件切分参数 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--设置日志基础文件大小，超过该大小就触发日志文件滚动更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--设置日志文件滚动更新的时间，依赖于文件命名filePattern的设置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--设置日志的文件个数上限，不设置默认为7个，超过大小后会被覆盖；依赖于filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--debug级别日志-->
        <RollingFile name="debugFileAppender"
                     fileName="${LOG_HOME}/debug.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/debug-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--过滤掉info及更高级别日志-->
                <ThresholdFilter level="info" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--设置日志格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 设置日志文件切分参数 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--设置日志基础文件大小，超过该大小就触发日志文件滚动更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--设置日志文件滚动更新的时间，依赖于文件命名filePattern的设置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--设置日志的文件个数上限，不设置默认为7个，超过大小后会被覆盖；依赖于filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--info级别日志-->
        <RollingFile name="infoFileAppender"
                     fileName="${LOG_HOME}/info.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--过滤掉warn及更高级别日志-->
                <ThresholdFilter level="warn" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--设置日志格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
            <!-- 设置日志文件切分参数 -->
            <!--<OnStartupTriggeringPolicy/>-->
            <!--设置日志基础文件大小，超过该大小就触发日志文件滚动更新-->
            <SizeBasedTriggeringPolicy size="100 MB"/>
            <!--设置日志文件滚动更新的时间，依赖于文件命名filePattern的设置-->
            <TimeBasedTriggeringPolicy interval="1" modulate="true" />
            </Policies>
            <!--设置日志的文件个数上限，不设置默认为7个，超过大小后会被覆盖；依赖于filePattern中的%i-->
            <!--<DefaultRolloverStrategy max="100"/>-->
        </RollingFile>

        <!--warn级别日志-->
        <RollingFile name="warnFileAppender"
                     fileName="${LOG_HOME}/warn.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--过滤掉error及更高级别日志-->
                <ThresholdFilter level="error" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--设置日志格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 设置日志文件切分参数 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--设置日志基础文件大小，超过该大小就触发日志文件滚动更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--设置日志文件滚动更新的时间，依赖于文件命名filePattern的设置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--设置日志的文件个数上限，不设置默认为7个，超过大小后会被覆盖；依赖于filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--error及更高级别日志-->
        <RollingFile name="errorFileAppender"
                     fileName="${LOG_HOME}/error.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz">
            <!--设置日志格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 设置日志文件切分参数 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--设置日志基础文件大小，超过该大小就触发日志文件滚动更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--设置日志文件滚动更新的时间，依赖于文件命名filePattern的设置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--设置日志的文件个数上限，不设置默认为7个，超过大小后会被覆盖；依赖于filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--json格式error级别日志-->
        <RollingFile name="errorJsonAppender"
                     fileName="${LOG_HOME}/error-json.log"
                     filePattern="${LOG_HOME}/error-json-%d{yyyy-MM-dd}-%i.log.gz">
            <JSONLayout compact="true" eventEol="true" locationInfo="true"/>
            <Policies>
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
            </Policies>
        </RollingFile>
    </Appenders>

    <Loggers>
        <!-- 根日志设置 -->
        <Root level="debug">
            <AppenderRef ref="allFileAppender" level="all"/>
            <AppenderRef ref="consoleAppender" level="debug"/>
            <AppenderRef ref="debugFileAppender" level="debug"/>
            <AppenderRef ref="infoFileAppender" level="info"/>
            <AppenderRef ref="warnFileAppender" level="warn"/>
            <AppenderRef ref="errorFileAppender" level="error"/>
            <AppenderRef ref="errorJsonAppender" level="error"/>
        </Root>

        <!--spring日志-->
        <Logger name="org.springframework" level="info"/>
        <!--druid数据源日志-->
        <Logger name="druid.sql.Statement" level="warn"/>
        <!-- mybatis日志 -->
        <Logger name="com.mybatis" level="warn"/>
    </Loggers>

</Configuration>
```

同样也可以通过appender和patternLayout等标签来设置日志的输出。

然后在application.properties中加上log4j2的配置

```properties
logging.config=classpath:log4j2.xml
```

测试一下，可以通过Lombok的`@Log4j2`注释使用日志

```java
@SpringBootApplication
@Log4j2
public class Log4j2UsageApplication {

    public static void main(String[] args) {
        SpringApplication.run(Log4j2UsageApplication.class, args);
        log.warn("this is form log4j2.");
    }
}
```

![](https://pics.findfuns.org/log4j2.png)

同时按照log4j2.xml的配置对应的目录下可以找到按照日志级别分类的记录

<img src="https://pics.findfuns.org/log4j2-files.png" style="zoom:50%;" />