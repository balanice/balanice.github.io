---
title: "slf4j + logback 记录日志"
date: 2021-01-24
tags:
    Java
    Springboot
layout: post
---

## 如何使用

Springboot 项目无需安装， 框架中已经集成了 slf4j 和 logback. 创建 `src/main/resource/logback.xml` 文件， 文件配置如下：

```xml
<configuration>
    <property name="LOG_HOME" value="logs"/>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} %line - %msg%n</pattern>
            <charset>utf-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/demo.log</file>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>"%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} %line - %msg%n"</pattern>
            <charset>utf-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>log/demo-%d{yyyy-MM-dd}_%i.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
        </rollingPolicy>
    </appender>

    <root level="debug">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```