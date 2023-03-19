# ELK搭建日志收集系统



## 1.概述

利用docker容器化以后，需要考虑如何采集位于Docker容器中的应用程序的打印日志供运维分析。比如SpringBoot应用的日志收集。利用ELK日志中心来收集容器化应用程序所产生的日志，并且可以用可视化的方式对日志进行查询与分析，其架构如下图所示

### 1.1 架构图

![架构图](ELK搭建日志收集系统.assets/a237eda0f1c441eab58ad5040d278f11_tplv-k3u1fbpfcp-zoom-1.image)

## 2  环境搭建

ELK 的安装，此处略过，不是重点。



## 3.SpringBoot应用集成

SpringBoot应用集成Logstash

```xml
<!--集成logstash-->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```

添加配置文件logback-spring.xml 让logback的日志输出到logstash

> 注意appender节点下的destination需要改成你自己的logstash服务地址，例如我自己的是:121.196.184.98:5044

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
    <!--应用名称-->
    <property name="APP_NAME" value="mall-admin"/>
    <!--日志文件保存路径-->
    <property name="LOG_FILE_PATH" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/logs}"/>
    <contextName>${APP_NAME}</contextName>

    <!--每天记录日志到文件appender-->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!--输出到logstash的appender-->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <!--可以访问的logstash日志收集端口-->
        <destination>39.103.203.41:4560</destination>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="LOGSTASH"/>
    </root>
</configuration>
```

运行SpringBoot应用

```java
package com.macro.mall.tiny;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MallApplication {

 public static void main(String[] args) {
  SpringApplication.run(MallApplication.class, args);
 }

}
```

在kibana中查看日志信息













