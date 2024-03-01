# 前言

作者目前打算分享一期关于devOps系列的文章，希望对热爱学习和探索的你有所帮助。

文章主要记录一些简洁、高效的运维部署指令，旨在 **记录**和能够**快速地构建系统**。就像运维文档或者手册一样，方便进行系统的重建、改造和优化。每篇文章独立出来，可以单独作为其中一项组件的部署和使用。

本章为 **devOps系列（八）efk+prometheus+grafana日志监控和告警**



## 大纲

[devOps系列介绍]()

[devOps系列（一）docker搭建]()

[devOps系列（二）gitlab搭建]()

[devOps系列（三）nexus-harbor搭建]()

[devOps系列（四）jenkins搭建]()

[devOps系列（五）efk系统搭建]()

[devOps系列（六）grafana+prometheus搭建]()

[devOps系列（七）grafana+prometheus监控告警]()

[devOps系列（八）efk+prometheus+grafana日志监控和告警]()





# 正文

## 日志收集

目前我们已经搭建好了efk日志系统，接下来就是把日志数据采集进来。

目前java程序的采集，可以在框架侧写一个基于logback日志收集starter依赖框架，便于日志收集的安装和管理。

可以自建一个starter依赖工程项目，也可以直接植入项目工程。

注：本文着重介绍核心原理，可能无法直接使用

需要引入的依赖

```
  <dependency>
            <groupId>com.sndyuk</groupId>
            <artifactId>logback-more-appenders</artifactId>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>
        <dependency>
            <groupId>org.komamitsu</groupId>
            <artifactId>fluency-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.komamitsu</groupId>
            <artifactId>fluency-fluentd</artifactId>
        </dependency>

```



核心logback配置文件  logback-spring.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <!--定义日志文件的存储地址 勿在 LogBack 的配置中使用相对路径-->
    <springProperty name="profile" source="spring.profiles.active"/>
    <springProperty name="applicationName" source="spring.application.name"/>
    <!--   默认地址 -->
    <springProperty name="fluentdAddr" source="framework.logback.fluentd-addr" defaultValue="fluentd.jafir.top"/>
    <property name="LOG_HOME" value="/${applicationName}/logs"/>
    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <!-- info及其以上日志 -->
    <appender name="LOCAL_ALL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_HOME}/info_log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <!-- 设置编码格式，以防中文乱码 -->
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
        <!--日志文件最大的大小-->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>10MB</MaxFileSize>
        </triggeringPolicy>
    </appender>

    <!-- 错误日志 -->
    <appender name="LOCAL_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_HOME}/error_log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <!-- 设置编码格式，以防中文乱码 -->
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
        <!--日志文件最大的大小-->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>10MB</MaxFileSize>
        </triggeringPolicy>
    </appender>

    <!-- Fluency -->
    <appender name="FLUENCY_SYNC" class="ch.qos.logback.more.appenders.FluencyLogbackAppender">
        <!-- Tag for Fluentd. Farther information: http://docs.fluentd.org/articles/config-file -->
        <!-- 微服务名 -->
        <tag>${applicationName}</tag>
        <!-- [Optional] Label for Fluentd. Farther information: http://docs.fluentd.org/articles/config-file -->

        <!-- Host name/address and port number which Fluentd placed -->
        <remoteHost>${fluentdAddr}</remoteHost>
        <port>24224</port>

        <!-- [Optional] Multiple name/addresses and port numbers which Fluentd placed
       <remoteServers>
          <remoteServer>
            <host>primary</host>
            <port>24224</port>
          </remoteServer>
          <remoteServer>
            <host>secondary</host>
            <port>24224</port>
          </remoteServer>
        </remoteServers>
         -->

        <!-- [Optional] Additional fields(Pairs of key: value) -->
        <!-- 环境 -->
        <additionalField>
            <key>env</key>
            <value>${profile}</value>
        </additionalField>

        <!-- [Optional] Configurations to customize Fluency's behavior: https://github.com/komamitsu/fluency#usage  -->
        <ackResponseMode>false</ackResponseMode>
        <!-- <fileBackupDir>/tmp</fileBackupDir> -->
        <bufferChunkInitialSize>33554432</bufferChunkInitialSize>
        <bufferChunkRetentionSize>268435456</bufferChunkRetentionSize>
        <maxBufferSize>1073741824</maxBufferSize>
        <bufferChunkRetentionTimeMillis>1000</bufferChunkRetentionTimeMillis>
        <connectionTimeoutMilli>5000</connectionTimeoutMilli>
        <readTimeoutMilli>5000</readTimeoutMilli>
        <waitUntilBufferFlushed>30</waitUntilBufferFlushed>
        <waitUntilFlusherTerminated>40</waitUntilFlusherTerminated>
        <flushAttemptIntervalMillis>200</flushAttemptIntervalMillis>
        <senderMaxRetryCount>12</senderMaxRetryCount>
        <!-- [Optional] Enable/Disable use of EventTime to get sub second resolution of log event date-time -->
        <useEventTime>true</useEventTime>
        <sslEnabled>false</sslEnabled>
        <!-- [Optional] Enable/Disable use the of JVM Heap for buffering -->
        <jvmHeapBufferMode>false</jvmHeapBufferMode>
        <!-- [Optional] If true, Map Marker is expanded instead of nesting in the marker name -->
        <flattenMapMarker>false</flattenMapMarker>
        <!--  [Optional] default "marker" -->
        <markerPrefix></markerPrefix>

        <!-- [Optional] Message encoder if you want to customize message -->
        <encoder>
            <pattern><![CDATA[%-5level %logger{50}#%line %message]]></pattern>
        </encoder>

        <!-- [Optional] Message field key name. Default: "message" -->
        <messageFieldKeyName>msg</messageFieldKeyName>

    </appender>

    <!-- Fluency -->
    <appender name="FLUENCY_SYNC_ACCESS" class="ch.qos.logback.more.appenders.FluencyLogbackAppender">
        <!-- Tag for Fluentd. Farther information: http://docs.fluentd.org/articles/config-file -->
        <!-- 微服务名 -->
        <tag>access-${applicationName}</tag>
        <!-- [Optional] Label for Fluentd. Farther information: http://docs.fluentd.org/articles/config-file -->

        <!-- Host name/address and port number which Fluentd placed -->
        <remoteHost>${fluentdAddr}</remoteHost>
        <port>24224</port>

        <!-- [Optional] Multiple name/addresses and port numbers which Fluentd placed
       <remoteServers>
          <remoteServer>
            <host>primary</host>
            <port>24224</port>
          </remoteServer>
          <remoteServer>
            <host>secondary</host>
            <port>24224</port>
          </remoteServer>
        </remoteServers>
         -->

        <!-- [Optional] Additional fields(Pairs of key: value) -->
        <!-- 环境 -->
        <additionalField>
            <key>env</key>
            <value>${profile}</value>
        </additionalField>

        <!-- [Optional] Configurations to customize Fluency's behavior: https://github.com/komamitsu/fluency#usage  -->
        <ackResponseMode>false</ackResponseMode>
        <!-- <fileBackupDir>/tmp</fileBackupDir> -->
        <bufferChunkInitialSize>33554432</bufferChunkInitialSize>
        <bufferChunkRetentionSize>268435456</bufferChunkRetentionSize>
        <maxBufferSize>1073741824</maxBufferSize>
        <bufferChunkRetentionTimeMillis>1000</bufferChunkRetentionTimeMillis>
        <connectionTimeoutMilli>5000</connectionTimeoutMilli>
        <readTimeoutMilli>5000</readTimeoutMilli>
        <waitUntilBufferFlushed>30</waitUntilBufferFlushed>
        <waitUntilFlusherTerminated>40</waitUntilFlusherTerminated>
        <flushAttemptIntervalMillis>200</flushAttemptIntervalMillis>
        <senderMaxRetryCount>12</senderMaxRetryCount>
        <!-- [Optional] Enable/Disable use of EventTime to get sub second resolution of log event date-time -->
        <useEventTime>true</useEventTime>
        <sslEnabled>false</sslEnabled>
        <!-- [Optional] Enable/Disable use the of JVM Heap for buffering -->
        <jvmHeapBufferMode>false</jvmHeapBufferMode>
        <!-- [Optional] If true, Map Marker is expanded instead of nesting in the marker name -->
        <flattenMapMarker>false</flattenMapMarker>
        <!--  [Optional] default "marker" -->
        <markerPrefix></markerPrefix>

        <!-- [Optional] Message encoder if you want to customize message -->
        <encoder>
            <pattern>%message%n</pattern>
        </encoder>

        <!-- [Optional] Message field key name. Default: "message" -->
        <messageFieldKeyName>msg</messageFieldKeyName>

    </appender>

    <appender name="FLUENCY" class="ch.qos.logback.classic.AsyncAppender">
        <!-- Max queue size of logs which is waiting to be sent (When it reach to the max size, the log will be disappeared). -->
        <queueSize>999</queueSize>
        <!-- Never block when the queue becomes full. -->
        <neverBlock>true</neverBlock>
        <!-- The default maximum queue flush time allowed during appender stop.
             If the worker takes longer than this time it will exit, discarding any remaining items in the queue.
             10000 millis
         -->
        <maxFlushTime>1000</maxFlushTime>
        <appender-ref ref="FLUENCY_SYNC"/>
    </appender>

    <appender name="FLUENCY_ACCESS" class="ch.qos.logback.classic.AsyncAppender">
        <!-- Max queue size of logs which is waiting to be sent (When it reach to the max size, the log will be disappeared). -->
        <queueSize>999</queueSize>
        <!-- Never block when the queue becomes full. -->
        <neverBlock>true</neverBlock>
        <!-- The default maximum queue flush time allowed during appender stop.
             If the worker takes longer than this time it will exit, discarding any remaining items in the queue.
             10000 millis
         -->
        <maxFlushTime>1000</maxFlushTime>
        <appender-ref ref="FLUENCY_SYNC_ACCESS"/>
    </appender>

    <springProfile name="local">
        <!-- 日志输出级别 -->
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
            <appender-ref ref="FLUENCY"/>
            <!--            <appender-ref ref="LOCAL_ALL"/>-->
            <!--            <appender-ref ref="LOCAL_ERROR"/>-->
            <!--            <appender-ref ref="FLUENCY"/>-->
        </root>
        <logger name="com.jafir.logback.aop.WebLogAspect" level="INFO" additivity="false">
            <appender-ref ref="STDOUT"/>
            <!--            <appender-ref ref="FLUENCY_ACCESS"/>-->
        </logger>
    </springProfile>

    <springProfile name="dev,test,preprod">
        <!-- 日志输出级别 -->
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
            <!--            <appender-ref ref="LOCAL_ALL"/>-->
            <!--            <appender-ref ref="LOCAL_ERROR"/>-->
            <appender-ref ref="FLUENCY"/>
        </root>
        <logger name="com.jafir.logback.aop.WebLogAspect" level="INFO" additivity="false">
            <appender-ref ref="STDOUT"/>
            <appender-ref ref="FLUENCY_ACCESS"/>
        </logger>
    </springProfile>

    <springProfile name="prod">
        <!-- 日志输出级别 -->
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
<!--            <appender-ref ref="LOCAL_ALL"/>-->
<!--            <appender-ref ref="LOCAL_ERROR"/>-->
            <appender-ref ref="FLUENCY"/>
        </root>
        <logger name="com.jafir.logback.aop.WebLogAspect" level="INFO" additivity="false">
            <appender-ref ref="STDOUT"/>
            <appender-ref ref="FLUENCY_ACCESS"/>
        </logger>
    </springProfile>

    <!-- 关闭某个日志打印 -->
    <logger name="org.komamitsu.fluency.Fluency" level="OFF" />
    <logger name="org.komamitsu.fluency.fluentd.ingester.sender.RetryableSender" level="OFF" />
    <logger name="org.komamitsu.fluency.fluentd.ingester.sender.NetworkSender" level="OFF" />

</configuration>

```



springBoot的AutoConfiguration类

```
package com.jafir.logback;

import com.jafir.logback.aop.WebLogAspect;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;


@Configuration
@Import(WebLogAspect.class)
@EnableConfigurationProperties(LogbackProperties.class)
@ConditionalOnProperty(prefix = "framework.logback", value = "enabled", havingValue = "true", matchIfMissing = true)
@ComponentScan(value = "com.jafir.logback")
public class LogbackAutoConfiguration {
}
```

```
package com.jafir.logback;

import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.List;

@ConfigurationProperties("framework.logback")
public class LogbackProperties {
    private Boolean enabled = false;
    private String fluentdAddr = "fluentd.jaifr.top";
    private List<String> excludeUrl;

    public List<String> getExcludeUrl() {
        return excludeUrl;
    }

    public void setExcludeUrl(List<String> excludeUrl) {
        this.excludeUrl = excludeUrl;
    }

    public Boolean getEnabled() {
        return enabled;
    }

    public void setEnabled(Boolean enabled) {
        this.enabled = enabled;
    }

    public String getFluentdAddr() {
        return fluentdAddr;
    }

    public void setFluentdAddr(String fluentdAddr) {
        this.fluentdAddr = fluentdAddr;
    }
}
```

可以通过yml配置文件来进行装配控制

> framework.logback.enabled  控制是否开启日志收集
>
> framework.logback.fluentdAddr 设置fluentd的地址
>
> framework.logback.excludeUrl 设置过滤不进行收集的地址



核心servlet拦截器类

```
package com.jafir.logback.aop;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.http.HttpStatus;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.gson.Gson;
import com.jafir.logback.LogResponseBody;
import com.jafir.logback.LogbackProperties;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.IOUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.web.servlet.HandlerMapping;
import org.springframework.web.util.ContentCachingResponseWrapper;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.*;

@Aspect
@Slf4j
public class WebLogAspect {
    private final boolean NEED_RESPONSE_BODY = true;


    private final LogbackProperties logbackProperties;
    private final ObjectMapper objectMapper;


    private static final List<String> DEFAULT_EXCLUDE_URL = new ArrayList<>();

    static {
        DEFAULT_EXCLUDE_URL.add("/actuator/prometheus");
        DEFAULT_EXCLUDE_URL.add("/health/detect");
    }

    public WebLogAspect(ObjectMapper objectMapper, LogbackProperties logbackProperties) {
        this.objectMapper = objectMapper;
        this.logbackProperties = logbackProperties;

        if (CollUtil.isNotEmpty(logbackProperties.getExcludeUrl())) {
            logbackProperties.getExcludeUrl().addAll(DEFAULT_EXCLUDE_URL);
        } else {
            logbackProperties.setExcludeUrl(DEFAULT_EXCLUDE_URL);
        }
    }

    @Around("execution(public void javax.servlet.http.HttpServlet.service(..)))")
    public Object webLog(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        DelegateHttpRequest servletRequest = new DelegateHttpRequest((HttpServletRequest) args[0]);
        HttpServletResponse servletResponse = (HttpServletResponse) args[1];
        ContentCachingResponseWrapper responseWrapper = null;
        if (NEED_RESPONSE_BODY) {
            responseWrapper = new ContentCachingResponseWrapper(servletResponse);
        }


        // 过滤不需要拦截的请求
        if (doNotIntercept(servletRequest)) {
            return joinPoint.proceed();
        }


        WebLog webLog = new WebLog();
        webLog.setTimestamp(Instant.now());

        WebLog.Request request = new WebLog.Request();

        InputStream servletRequestStream = servletRequest.getInputStream();

        int size;
        byte[] buffer = new byte[1024];
        ByteArrayOutputStream tmpRequestStream = new ByteArrayOutputStream();
        while ((size = servletRequestStream.read(buffer)) != -1) {
            tmpRequestStream.write(buffer, 0, size);
        }

        request.setBody(tmpRequestStream.toString());

        Map<String, List<String>> requestHeaders = new HashMap<>();

        Enumeration<String> servletRequestHeaders = servletRequest.getHeaderNames();
        while (servletRequestHeaders.hasMoreElements()) {
            String header = servletRequestHeaders.nextElement();
            Enumeration<String> values = servletRequest.getHeaders(header);
            List<String> list = new ArrayList<>();
            while (values.hasMoreElements()) {
                String value = values.nextElement();
                list.add(value);
            }
            requestHeaders.put(header, list);
        }
        request.setHeaders(requestHeaders);

        request.setMethod(servletRequest.getMethod());

        Object rawUrl = servletRequest.getAttribute("raw-api-uri");
        if (rawUrl instanceof String) {
            request.setRequestUri((String) rawUrl);
        } else {
            request.setRequestUri(servletRequest.getRequestURI());
        }

        Map<String, List<String>> parameters = new HashMap<>();
        for (Map.Entry<String, String[]> entry : servletRequest.getParameterMap().entrySet()) {
            List<String> list = new ArrayList<>(Arrays.asList(entry.getValue()));
            parameters.put(entry.getKey(), list);
        }
        request.setParameters(parameters);

        Object attributeStart = servletRequest.getAttribute("raw-api-start");
        long start;

        if (attributeStart instanceof Long) {
            start = (long) attributeStart;
        } else {
            start = System.nanoTime();
        }

        Object value;
        try {
            if (NEED_RESPONSE_BODY) {
                value = joinPoint.proceed(new Object[]{servletRequest, responseWrapper});
            } else {
                value = joinPoint.proceed(new Object[]{servletRequest, servletResponse});
            }
        } catch (Throwable e) {
            ((HttpServletRequest) args[0]).setAttribute("raw-api-uri", servletRequest.getRequestURI());
            ((HttpServletRequest) args[0]).setAttribute("raw-api-start", start);
            throw e;
        }

        @SuppressWarnings("unchecked")
        Map<String, String> pathMap = (Map<String, String>) servletRequest.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);

        if (pathMap != null && !pathMap.isEmpty()) {
            //写入path数据
            request.setPathParameters(pathMap);
        }

        long timeTaken = (System.nanoTime() - start) / 1_000_000;


        WebLog.Response response = new WebLog.Response();
        int status = servletResponse.getStatus();
        if (NEED_RESPONSE_BODY) {
            boolean isSuccess = true;
            // 成功的接口不用记录 responseBody
            if (HttpStatus.HTTP_OK != status) {
                isSuccess = false;
            }
            String responseBodyStr;
            try {
                responseBodyStr = IOUtils.toString(responseWrapper.getContentInputStream(), StandardCharsets.UTF_8.displayName());
            } catch (Exception e) {
                responseBodyStr = "";
                isSuccess = false;
                log.error("接口: {} ,IOUtils.toString 出现异常", servletRequest.getRequestURI());
            }
         
            // 失败的记录一下body
            if (!isSuccess) {
                response.setResponseBody(responseBodyStr);
            }
            try {
                responseWrapper.copyBodyToResponse();
            } catch (Exception e) {
                log.error("接口: {} ,copyBodyToResponse 出现异常", servletRequest.getRequestURI());
            }
        }
        response.setStatus(status);
        Map<String, List<String>> responseHeaders = new HashMap<>();

        Collection<String> servletResponseHeaders = servletResponse.getHeaderNames();
        for (String headerName : servletResponseHeaders) {
            Collection<String> values = servletResponse.getHeaders(headerName);
            List<String> list = new ArrayList<>(values);
            responseHeaders.put(headerName, list);
        }
        response.setHeaders(responseHeaders);

        String bestUri = String.valueOf(servletRequest.getRequest().getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE));
        //兼容处理    抛异常情况下该值null 用requestUri兼容
        if(bestUri!=null && !bestUri.isEmpty() && !"null".equals(bestUri)) {
            request.setUri(bestUri);
        }else {
            request.setUri(request.getRequestUri());
        }

        webLog.setTimeTaken(timeTaken);
        webLog.setRequest(request);
        webLog.setResponse(response);

        log.info(objectMapper.writeValueAsString(webLog));

        return value;
    }


    private boolean doNotIntercept(DelegateHttpRequest servletRequest) {

        // 放行文件类型
        if (servletRequest.getContentType() != null && servletRequest.getContentType().contains("multipart")) {
            return true;
        }

        // 如果是 post的 x-www-form 类型 也就是 xx=xx&xx=xx&xx 这种格式的 （一般很少有这样使用的）
        if ((servletRequest.getContentType() != null
                && servletRequest.getContentType().contains("application/x-www-form-urlencoded")
                && "post".equalsIgnoreCase(servletRequest.getMethod()))) {
            return true;
        }

        // 放行不做拦截的uri
        for (String uri : logbackProperties.getExcludeUrl()) {
            if (uri.equals(servletRequest.getRequestURI())) {
                return true;
            }
        }

        return false;
    }
}
```



日志bean

```
package com.jafir.logback.aop;

import lombok.Data;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@Data
public class WebLog {
    private Instant timestamp;
    private Long timeTaken;
    private Request request;
    private Response response;

    @Data
    public static class Request {
        private String method;
        private String uri;
        private String requestUri;
        private Map<String, List<String>> headers;
        private Map<String, List<String>> parameters;
        private String body;
        private Map<String, String> pathParameters;
    }

    @Data
    public static class Response {
        /**
         * http 的 status
         */
        private Integer status;
        /**
         * WebResponseBody 的 code
         */
        private Integer bodyCode;
        private Map<String, List<String>> headers;
        private String responseBody;
    }
}
```

request的代理类（主要目的是保留读取到的流数据。流只能读取一次）

```
package com.jafir.logback.aop;

import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;

public class DelegateHttpRequest extends HttpServletRequestWrapper {
    private byte[] bytes;
    private final byte[] buffer = new byte[4096];

    public DelegateHttpRequest(HttpServletRequest request) {
        super(request);
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        if (bytes == null) {
            ServletInputStream inputStream = super.getInputStream();
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            int size = 0;
            while ((size = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, size);
            }
            outputStream.close();
            bytes = outputStream.toByteArray();
        }

        return new DelegateServletInputStream(new ByteArrayInputStream(bytes));
    }
}
```



```
package com.jafir.logback.aop;

import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import java.io.IOException;
import java.io.InputStream;

public class DelegateServletInputStream extends ServletInputStream {
    private final InputStream inputStream;

    public DelegateServletInputStream(InputStream inputStream){
        this.inputStream = inputStream;
    }

    @Override
    public boolean isFinished() {
        return false;
    }

    @Override
    public boolean isReady() {
        return true;
    }

    @Override
    public void setReadListener(ReadListener listener) { }

    @Override
    public int read() throws IOException {
        return inputStream.read();
    }
}
```



以上核心内容其实就是 一个拦截器 进行拦截接口，然后按照一定的结构打印日志，然后logback再利用appender 写入到fluentd中，完成日志的收集。



fluent的日志收集大致如下：



![fluency](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/fluency.png)







##### 收集的日志结构中比较重要的字段有：

> timetaken： 接口耗时
>
> request：请求
>
> response：返回结果（错误时包含body信息）
>
> status: 表示http的状态   正常都是200
>
> bodyCode:  表示websponse结构里面的code值（如果你的返回结构是 在http之上 又封装了一层 msg code body的话 这里的bodyCode就是 返回的结果里面的code，一般我们会对其进行业务异常和系统异常的code区分。比如 200是正常，500为系统异常，其他是业务异常等）



目前比较重要的是logback-spring.xml中，有几个点。

env :   用于区分环境，logback-spring.xml是支持 profile 获取的

FLUENCY_SYNC： 普通的日志收集，就是整个应用程序的日志。如果是应用日志 则索引名为 $applicationName-年月日 

FLUENCY_SYNC_ACCESS:   访问日志的收集，也就是接口的请求日志 通过webaspect拦截器写的日志。如果是访问日志 则索引名为access-$applicationName--年月日

这样的话可以在es中区分开应用日志和访问日志，FLUENCY_SYNC和FLUENCY_SYNC_ACCESS 也能够分开进行收集。

如： 普通的日志就用 FLUENCY_SYNC   ，只有 WebLogAspect 下面的拦截器日志，用FLUENCY_SYNC_ACCESS收集

```
 <springProfile name="prod">
        <!-- 日志输出级别 -->
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
<!--            <appender-ref ref="LOCAL_ALL"/>-->
<!--            <appender-ref ref="LOCAL_ERROR"/>-->
            <appender-ref ref="FLUENCY"/>
        </root>
        
        <logger name="com.jafir.logback.aop.WebLogAspect" level="INFO" additivity="false">
            <appender-ref ref="STDOUT"/>
            <appender-ref ref="FLUENCY_ACCESS"/>
        </logger>
    </springProfile>
```



如上则完成了 efk的日志收集，最终在kibana中可以通过新建pattern来查阅筛选日志信息。



## 日志监控和告警

对于服务的接口已经按照不同的索引存在于了es中，我们也可以用grafana来进行展示和监控。

#### grafana添加es datasource

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bbffe74be424.png)

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc075bafbce4.png)

#### 添加监控表

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc0c95daaa9b.png)

#### 错误统计

错误数query条件: 利用status 或者 bodeCode

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc1688c32340.png)

```
env:"test" AND @log_name:"access-xxxx" AND   !response.status:"200"
```

意为：测试环境下的xxx服务，返回结果不等于200的数量

#### 接口响应统计

接口响应query条件：  利用timeTaken

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc1c12874188.png)

```
env:"test" AND @log_name:"access-jisu-http-web"
```



###### 注意：

grafan的监控表 query不能使用变量，只能写死，所以可能会写多个环境 多个服务 多张表



### 添加告警

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc3ef691c8af.png)

告警理论上可以使用grafana自身集成的alertmanager 但是尝试之后发现并不好用 所以我们还是使用 前面prometheus监控搭建得 alertmanager 和 prometheus-alert结合使用

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc52ad99aab5.png)

这里就添加对应地址即可

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc5d6a7b8a2e.png)

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc64c5de3d47.png)

其他可以默认 然后就好了

### 原理介绍

> es数据源-》grafana （监控数据表 触发告警条件 发送告警） -》 alertmanager （配置路由到指定webhook） -》 prometheus-alert （根据不同模板组装数据）-》企业微信

### alertmanager和prometheus-alert配置调整

prometheus-alert地址

```
http://192.168.20.2:8080/
```

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc7d52a5a894.png)

找到grafana-wx 然后设置模板

```
{{range $k, $v := .alerts}}{{if eq $v.status "resolved"}}## [Prometheus恢复]()
###### 告警类型: {{$v.labels.alertname}}
###### 告警状态: {{ $v.status }}
###### 告警详情: {{$v.annotations.__value_string__}}
###### 故障时间：{{GetCSTtime $v.startsAt}}
###### 恢复时间：{{GetCSTtime $v.endsAt}}
{{else}}
## [Prometheus告警]()
###### 告警类型: {{$v.labels.alertname}}
###### 告警状态: {{ $v.status }}
###### 告警详情: {{$v.annotations.__value_string__}}
###### 故障时间：{{GetCSTtime $v.startsAt}}
{{end}}{{end}}
```

也可以进行测试 （测试内容可以从prometheus-alert日志中寻找）

```
{"receiver":"web\\.hook\\.grafanaalert","status":"resolved","alerts":[{"status":"resolved","labels":{"__alert_rule_namespace_uid__":"IrqNMj34z","__alert_rule_uid__":"lBw5-C3Vz","alertname":"DatasourceNoData","datasource_uid":"bP2dUr3Vz","ref_id":"A","rulename":"api-server错误"},"annotations":{"__dashboardUid__":"CBAou9qVz","__panelId__":"10"},"startsAt":"2023-08-03T00:00:12.197Z","endsAt":"2023-08-03T01:12:05.562Z","generatorURL":"http://localhost:3000/alerting/lBw5-C3Vz/edit","fingerprint":"f265175a34e6cc2e"}],"groupLabels":{"alertname":"DatasourceNoData"},"commonLabels":{"__alert_rule_namespace_uid__":"IrqNMj34z","__alert_rule_uid__":"lBw5-C3Vz","alertname":"DatasourceNoData","datasource_uid":"bP2dUr3Vz","ref_id":"A","rulename":"api-server错误"},"commonAnnotations":{"__dashboardUid__":"CBAou9qVz","__panelId__":"10"},"externalURL":"http://alertmanager:9093","version":"4","groupKey":"{}/{__alert_rule_namespace_uid__=\"IrqNMj34z\"}:{alertname=\"DatasourceNoData\"}","truncatedAlerts":0}
```

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bc97f3bc5d62.png)

#### prometheus告警模板

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bcca650db46d.png)

```
{{range $k, $v := .alerts}}{{if eq $v.status "resolved"}}
## [Prometheus恢复]()
###### 告警类型: {{$v.labels.alertname}}
###### 故障主机: {{$v.labels.instance}}
###### 环境类型：{{$v.labels.job}}
###### 告警详情: {{$v.annotations.description}}
###### 故障时间：{{GetCSTtime $v.startsAt}}
###### 恢复时间：{{GetCSTtime $v.endsAt}}{{else}}
## [Prometheus告警]()
###### 告警类型: {{$v.labels.alertname}}
###### 故障主机: {{$v.labels.instance}}
###### 环境类型：{{$v.labels.job}}
###### 告警详情: {{$v.annotations.description}}
###### 故障时间：{{GetCSTtime $v.startsAt}}{{end}}
{{end}}
```

#### alertmanager配置

```
global:
  resolve_timeout: 15s
route:
  group_by: ['alertname','instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 2m
  receiver: 'web.hook.prometheusalert'
  routes:
  - receiver: 'web.hook.grafanaalert'  # 路由到名为 "web.hook.grafanaalert" 的接收器
    match:
      __alert_rule_namespace_uid__: 'IrqNMj34z'  # 匹配 alertname 为 "grafana" 的告警
receivers:
- name: 'web.hook.prometheusalert'
  webhook_configs:
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=你的企业微信webhook'
- name: 'web.hook.grafanaalert'
  webhook_configs:
  - url: 'http://prometheus-alert:8080/prometheusalert?type=wx&tpl=grafana-wx&wxurl=你的企业微信webhook'
```

**配置含义：**

10s 检测一下 2m 再重复提示

默认情况下都认为是prometheus的告警，走prometheusalert发送到对应prometheus-wx的模板

如果是数据包含 `__alert_rule_namespace_uid__: 'IrqNMj34z'` 则认为是grafana的告警 走grafanaalert发送到对应grafana-wx的模板



以上配置可以自适应调整，如果有发短信 或者 打电话告警的，也可以利用prometheusAlert全家桶的方式接入进来。





##### 测验

配置好了之后 就可以在grafana进行告警测试了

![img](https://jafir-blog-oss.oss-cn-hangzhou.aliyuncs.com/2024/images/1777bd23951a5a47.png)
