---
title: Spring Boot 整合日誌
date: 2024-08-30 19:05:59
tags:
  - Spring Boot
  - 日誌
categories:
  - [Spring Boot, 日誌]
cover: https://pics.findfuns.org/springboot.png
---
## 前言

Java髮展至今已經形成了一套完整的日誌體繫，分爲日誌門麵和日誌具體實現。日誌門麵相當於一個接口，日誌各種實現其實就是日誌門麵的不同實現。

Java的日誌加載依賴於**SPI**，不同於API， SPI模式下接口處於調用者一側，開髮者一側隻擁有具體的實現。而API模式下接口和具體實現都處於開髮者一側，調用者無需關心具體實現，隻管調用開髮者提供的接口即可。

SPI機製給予了程序動態加載的能力，程序運行的過程中根據接口去動態掃描對應的接口實現類並加載。

舉個例子，Java的日誌門麵Slf4j對應的具體實現有Spring Boot自帶的也是默認的logback，還有log4j，log4j2等，在使用時會動態加載classpath中存在的slf4j對應的實現類並把它加載使用，這就是SPI機製。

<img src="https://pics.findfuns.org/java-log-structure.png" style="zoom:50%;" />

### Logback

logback是Spring Boot默認使用的日誌實現，即使不配置也可以直接使用。

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

也可以使用Lombok提供的注解`@Slf4j`來輸出log，這樣簡潔一些。

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

但是默認的配置還是不能夠更加精準的滿足各種需求，需要進行自定義設置。

在classpath下創建logback.xml或者logback-spring.xml

```xml
<!--
logback的日誌級別： ERROR > WARN > INFO > DEBUG > TRACE
-->
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <!-- 定義日誌文件名稱 -->
    <property name="APP_NAME" value="log-files" />
    <!-- 定義日誌文件的路徑 -->
    <property name="LOG_PATH" value="${user.home}/${APP_NAME}" />
    <!-- 定義日誌的文件名 -->
    <property name="LOG_FILE" value="${LOG_PATH}/redis-usage.log" />

    <!-- 滾動記錄日誌，先將日誌記錄到指定文件，當符合某個條件時，將日誌記錄到其他文件 -->
    <appender name="APPLICATION"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 指定日誌文件的名稱 -->
        <file>${LOG_FILE}</file>
        <!--
          當髮生滾動時，決定 RollingFileAppender 的行爲，涉及文件移動和重命名
          TimeBasedRollingPolicy： 最常用的滾動策略，它根據時間來製定滾動策略，既負責滾動也負責觸髮滾動。
          -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--
           滾動時産生的文件的存放位置及文件名稱
           %d{yyyy-MM-dd}：按天進行日誌滾動
           %i：當文件大小超過maxFileSize時，按照i進行文件滾動
           -->
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!--
           可選節點，控製保留的歸檔文件的最大數量，超出數量就刪除舊文件。假設設置每天滾動，
           且maxHistory是7，則隻保存最近7天的文件，刪除之前的舊文件。
           注意，刪除舊文件時，那些爲了歸檔而創建的目錄也會被刪除。
           -->
            <maxHistory>7</maxHistory>
            <!--
           當日誌文件超過maxFileSize指定的大小時，根據上麵提到的%i進行日誌文件滾動
           注意此處配置SizeBasedTriggeringPolicy是無法實現按文件大小進行滾動的，
           必須配置timeBasedFileNamingAndTriggeringPolicy
           -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>50MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <!-- 日誌輸出格式： -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [ %thread ] - [ %-5level ] [ %logger{50} : %line ] - %msg%n</pattern>
        </layout>
    </appender>
    <!-- ch.qos.logback.core.ConsoleAppender 表示控製颱輸出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--
       日誌輸出格式：
           %d表示日期時間，%green 綠色
           %thread表示線程名，%magenta 洋紅色
           %-5level：級別從左顯示5個字符寬度 %highlight 高亮色
           %logger{36} 表示logger名字最長36個字符，否則按照句點分割 %yellow 黃色
           %msg：日誌消息
           %n是換行符
       -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%green(%d{yyyy-MM-dd HH:mm:ss.SSS}) [%magenta(%thread)] %highlight(%-5level) %yellow(%logger{36}): %msg%n</pattern>
        </layout>
    </appender>

    <!--
   root與logger是父子關繫，沒有特別定義則默認爲root，任何一個類隻會和一個logger對應，
   要麼是定義的logger，要麼是root，判斷的關鍵在於找到這個logger，然後判斷這個logger的appender和level。
   -->
    <root level="info">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="APPLICATION" />
    </root>
</configuration>
```

log的配置中有這麼幾個標籤

- property： 可以自定義變量名和變量值，方便拼接路徑等。
- appender： 每一個appender對應一個日誌的輸出地點
    - layout： 日誌的輸出格式
    - rollingPolicy： 日誌的滾動策略

通過這些標籤可以實現日誌的完全自定義。

使用logback時可以在application.yml或者application.properties中加入logback的配置，但也可以不配置，因爲Spring Boot的默認行爲是掃描classpath下是否有logback.xml或者logback-spring.xml。

```yaml
logging:
  config: classpath:logback-spring.xml
```

![](https://pics.findfuns.org/logback-customized.png)

同時日誌文件也可以在對應的目錄下找到

<img src="https://pics.findfuns.org/log-files.png" style="zoom:50%;" />

## Log4j2

log4j2和logback都是slf4j的具體實現，但在實際應用中二者不能同時存在，否則會報錯，導緻無法正常加載日誌的實現類。

所以在使用log4j2時需要在pom中顯示的剔除logback依賴，同時加入log4j2的依賴。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <!--剔除logback的依賴-->
  	<exclusions>
      <exclusion>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
      </exclusion>
  	</exclusions>
</dependency>
<!--log4j2依賴-->
	<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

在classpath下創建log4j2.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- log4j2的日誌級別： OFF > FATAL > ERROR > WARN > INFO > DEBUG > ALL -->
<Configuration>
    <!--<Configuration status="WARN" monitorInterval="30"> -->
    <properties>
        <property name="LOG_HOME">./service-logs</property>
    </properties>
    <Appenders>
        <!--*********************控製颱日誌***********************-->
        <Console name="consoleAppender" target="SYSTEM_OUT">
            <!--設置日誌格式及顏色-->
            <PatternLayout
                    pattern="%style{%d{ISO8601}}{bright,green} %highlight{%-5level} [%style{%t}{bright,blue}] %style{%C{}}{bright,yellow}: %msg%n%style{%throwable}{red}"
                    disableAnsi="false" noConsoleNoAnsi="false"/>
        </Console>

        <!--*********************文件日誌***********************-->
        <!--all級別日誌-->
        <RollingFile name="allFileAppender"
                     fileName="${LOG_HOME}/all.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/all-%d{yyyy-MM-dd}-%i.log.gz">
            <!--設置日誌格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 設置日誌文件切分參數 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--設置日誌基礎文件大小，超過該大小就觸髮日誌文件滾動更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--設置日誌文件滾動更新的時間，依賴於文件命名filePattern的設置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--設置日誌的文件個數上限，不設置默認爲7個，超過大小後會被覆蓋；依賴於filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--debug級別日誌-->
        <RollingFile name="debugFileAppender"
                     fileName="${LOG_HOME}/debug.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/debug-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--過濾掉info及更高級別日誌-->
                <ThresholdFilter level="info" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--設置日誌格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 設置日誌文件切分參數 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--設置日誌基礎文件大小，超過該大小就觸髮日誌文件滾動更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--設置日誌文件滾動更新的時間，依賴於文件命名filePattern的設置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--設置日誌的文件個數上限，不設置默認爲7個，超過大小後會被覆蓋；依賴於filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--info級別日誌-->
        <RollingFile name="infoFileAppender"
                     fileName="${LOG_HOME}/info.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--過濾掉warn及更高級別日誌-->
                <ThresholdFilter level="warn" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--設置日誌格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
            <!-- 設置日誌文件切分參數 -->
            <!--<OnStartupTriggeringPolicy/>-->
            <!--設置日誌基礎文件大小，超過該大小就觸髮日誌文件滾動更新-->
            <SizeBasedTriggeringPolicy size="100 MB"/>
            <!--設置日誌文件滾動更新的時間，依賴於文件命名filePattern的設置-->
            <TimeBasedTriggeringPolicy interval="1" modulate="true" />
            </Policies>
            <!--設置日誌的文件個數上限，不設置默認爲7個，超過大小後會被覆蓋；依賴於filePattern中的%i-->
            <!--<DefaultRolloverStrategy max="100"/>-->
        </RollingFile>

        <!--warn級別日誌-->
        <RollingFile name="warnFileAppender"
                     fileName="${LOG_HOME}/warn.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--過濾掉error及更高級別日誌-->
                <ThresholdFilter level="error" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--設置日誌格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 設置日誌文件切分參數 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--設置日誌基礎文件大小，超過該大小就觸髮日誌文件滾動更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--設置日誌文件滾動更新的時間，依賴於文件命名filePattern的設置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--設置日誌的文件個數上限，不設置默認爲7個，超過大小後會被覆蓋；依賴於filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--error及更高級別日誌-->
        <RollingFile name="errorFileAppender"
                     fileName="${LOG_HOME}/error.log"
                     filePattern="${LOG_HOME}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz">
            <!--設置日誌格式-->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 設置日誌文件切分參數 -->
                <!--<OnStartupTriggeringPolicy/>-->
                <!--設置日誌基礎文件大小，超過該大小就觸髮日誌文件滾動更新-->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!--設置日誌文件滾動更新的時間，依賴於文件命名filePattern的設置-->
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!--設置日誌的文件個數上限，不設置默認爲7個，超過大小後會被覆蓋；依賴於filePattern中的%i-->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>

        <!--json格式error級別日誌-->
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
        <!-- 根日誌設置 -->
        <Root level="debug">
            <AppenderRef ref="allFileAppender" level="all"/>
            <AppenderRef ref="consoleAppender" level="debug"/>
            <AppenderRef ref="debugFileAppender" level="debug"/>
            <AppenderRef ref="infoFileAppender" level="info"/>
            <AppenderRef ref="warnFileAppender" level="warn"/>
            <AppenderRef ref="errorFileAppender" level="error"/>
            <AppenderRef ref="errorJsonAppender" level="error"/>
        </Root>

        <!--spring日誌-->
        <Logger name="org.springframework" level="info"/>
        <!--druid數據源日誌-->
        <Logger name="druid.sql.Statement" level="warn"/>
        <!-- mybatis日誌 -->
        <Logger name="com.mybatis" level="warn"/>
    </Loggers>

</Configuration>
```

同樣也可以通過appender和patternLayout等標籤來設置日誌的輸出。

然後在application.properties中加上log4j2的配置

```properties
logging.config=classpath:log4j2.xml
```

測試一下，可以通過Lombok的`@Log4j2`注釋使用日誌

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

同時按照log4j2.xml的配置對應的目錄下可以找到按照日誌級別分類的記錄

<img src="https://pics.findfuns.org/log4j2-files.png" style="zoom:50%;" />