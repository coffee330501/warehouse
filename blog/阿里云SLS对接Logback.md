## 日志上传工具类

### 添加依赖

```xml
            <!-- sls -->
            <dependency>
                <groupId>com.aliyun.openservices</groupId>
                <artifactId>aliyun-log</artifactId>
                <version>0.6.35</version>
                <classifier>jar-with-dependencies</classifier>
            </dependency>
            <dependency>
                <groupId>com.aliyun.openservices</groupId>
                <artifactId>aliyun-log-producer</artifactId>
                <version>0.3.10</version>
            </dependency>
            <dependency>
                <groupId>com.aliyun.openservices</groupId>
                <artifactId>aliyun-log</artifactId>
                <version>0.6.33</version>
            </dependency>
            <dependency>
                <groupId>com.google.protobuf</groupId>
                <artifactId>protobuf-java</artifactId>
                <version>2.5.0</version>
            </dependency>
```

```java
import com.alibaba.fastjson2.JSONObject;
import com.aliyun.openservices.aliyun.log.producer.LogProducer;
import com.aliyun.openservices.aliyun.log.producer.Producer;
import com.aliyun.openservices.aliyun.log.producer.ProducerConfig;
import com.aliyun.openservices.aliyun.log.producer.ProjectConfig;
import com.aliyun.openservices.aliyun.log.producer.errors.ProducerException;
import com.aliyun.openservices.log.common.LogItem;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.BufferedReader;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

@Slf4j
@Component
@ConfigurationProperties(prefix = "aliyun.sls")
public class SlsLogUtil {
    private static String accessKeyId;
    private static String accessKeySecret;
    private static String endpoint;
    private static String project;
    private static String logStore;

    public static void logReq(HttpServletRequest request, HttpServletResponse response) {
        logReq(request, response, null);
    }

    public static void logReq(HttpServletRequest request, HttpServletResponse response, Exception exception) {
        try {
            // 解决nginx转发请求后ip变化的问题，需要现在nginx中配置x-forwarded-for
            String remoteAddr = request.getHeader("x-forwarded-for");
            String httpMethod = request.getMethod();
            StringBuffer requestURL = request.getRequestURL();
            Map<String, String[]> parameterMap = request.getParameterMap();
            Long requestStartTime = (Long) request.getAttribute("requestStartTime");
            int status = response.getStatus();

            Map<String, Object> map = new HashMap<>();
            map.put("url", requestURL);
            map.put("remoteAddr", remoteAddr);
            map.put("method", httpMethod);
            map.put("params", parameterMap);
            map.put("status", status);

            // 请求耗时
            map.put("ResponseTime", requestStartTime == null ? 0 : System.currentTimeMillis() - requestStartTime);

            //获取请求体数据
            BufferedReader reader = request.getReader();
            String line;
            StringBuilder sb = new StringBuilder();
            while ((line = reader.readLine()) != null) {
                sb.append(line);
            }
            map.put("body", sb.toString());

            // 异常信息
            if (exception != null) {
                HashMap<String, Object> expMap = new HashMap<>();
                expMap.put("message", exception.getMessage());
                expMap.put("stackTrace", exception.getStackTrace());
                map.put("exception", expMap);
            }

            // 上传日志
            sendLog(map, "REQUEST");
        } catch (Exception e) {
            log.error("SLS ERROR:", e);
        }
    }

    public static void sendLog(Map<String, Object> map, String level) {
        Producer producer = null;
        try {
            map.put("level", level);

            // 设置producer参数
            ProducerConfig producerConfig = new ProducerConfig();
            producerConfig.setBatchSizeThresholdInBytes(3 * 1024 * 1024);
            producerConfig.setBatchCountThreshold(10000);
            producerConfig.setIoThreadCount(10);
            producerConfig.setRetries(3);
            producer = new LogProducer(producerConfig);
            producer.putProjectConfig(new ProjectConfig(project, endpoint, accessKeyId, accessKeySecret));

            // 构建日志
            LogItem logItem = new LogItem();
            Set<String> keys = map.keySet();
            for (String key : keys) {
                Object value = map.get(key);
                String valueStr = "";
                if (value != null) {
                    valueStr = JSONObject.toJSONString(value);
                }
                logItem.PushBack(key, valueStr);
            }

            // 发送日志 TOPIC换成自己的
            producer.send(
                    project,
                    logStore,
                    "BMS",
                    "REQUEST",
                    logItem);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ProducerException e) {
            throw new RuntimeException(e);
        } finally {
            if (producer != null) {
                try {
                    producer.close();
                } catch (InterruptedException | ProducerException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public void setAccessKeyId(String accessKeyId) {
        SlsLogUtil.accessKeyId = accessKeyId;
    }

    public void setAccessKeySecret(String accessKeySecret) {
        SlsLogUtil.accessKeySecret = accessKeySecret;
    }

    public void setEndpoint(String endpoint) {
        SlsLogUtil.endpoint = endpoint;
    }

    public void setProject(String project) {
        SlsLogUtil.project = project;
    }

    public void setLogStore(String logStore) {
        SlsLogUtil.logStore = logStore;
    }

    public String getAccessKeyId() {
        return accessKeyId;
    }

    public String getAccessKeySecret() {
        return accessKeySecret;
    }

    public String getEndpoint() {
        return endpoint;
    }

    public String getProject() {
        return project;
    }

    public String getLogStore() {
        return logStore;
    }
}
```

## 异常上传

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.validation.BindException;
import org.springframework.validation.FieldError;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * 全局异常处理器
 *
 * @author qybh
 */
@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

 
    /**
     * 业务异常
     */
    @ExceptionHandler(ServiceException.class)
    public Map<String, Object> handleServiceException(ServiceException e, HttpServletRequest request, HttpServletResponse response) {
        log.error(e.getMessage(), e);
        Integer code = e.getCode();
        return buildResult(code, e.getMessage(), request, response, e);
    }
    
	// ...

    private Map<String, Object> buildResult(String message, HttpServletRequest request, HttpServletResponse response, Exception e) {
        return buildResult(500, message, request, response, e);
    }

    private Map<String, Object> buildResult(Integer code, String message, HttpServletRequest request, HttpServletResponse response, Exception e) {
        // 上传阿里云
        SlsLogUtil.logReq(request, response, e);
        HashMap<String, Object> res = new HashMap<>();
        res.put("code", code);
        res.put("message", message);
        res.put("msg", message);
        return res;
    }
}

```



## 记录请求进出口

```java
@Component
public class SlsLogInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 记录请求开始时间
        request.setAttribute("requestStartTime",System.currentTimeMillis());
        return HandlerInterceptor.super.preHandle(request, response, handler);
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        SlsLogUtil.logReq(request,response);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```

mvc配置注册拦截器

...

**注意：** request重复读取会报错，因此我们需要实现`HttpServletRequestWrapper`重写`getInputStream()`和`getReader()`方法，将请求参数body复制到自己requestWrapper中， 后续只操作自己的requestWrapper。

1. 自定义wrapper复制请求流， 如果不重写会报 java.lang.IllegalStateException: getInputStream() has already been called for this request 异常， 原因是request请求流不能重复读取。

```java
import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStreamReader;

public class RequestWrapper extends HttpServletRequestWrapper {
    public final String body;

    public RequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        StringBuilder buf = new StringBuilder();
        BufferedReader bufferedReader = request.getReader();
        String line;
        while ((line = bufferedReader.readLine()) != null) {
            buf.append(line);
        }
        body = buf.toString();

    }

    public String getBody() {
        return body;
    }

    @Override
    public ServletInputStream getInputStream() {
        final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(body.getBytes());
        return new ServletInputStream() {
            @Override
            public int read() {
                return byteArrayInputStream.read();
            }

            @Override
            public boolean isFinished() {
                return false;
            }

            @Override
            public boolean isReady() {
                return false;
            }

            @Override
            public void setReadListener(ReadListener listener) {

            }
        };
    }

    @Override
    public BufferedReader getReader() {
        return new BufferedReader(new InputStreamReader(this.getInputStream()));
    }
}
```



2. 将自定义的wrapper通过过滤器传下去， 不传不会调用重写后的getInputStream()和getReader()方法

```java

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.Objects;
import java.util.Optional;

@WebFilter(urlPatterns = "/*", filterName = "channelFilter")
@Component
public class ChannelFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws ServletException, IOException {
        //文件上传类型 不需要处理，否则会报java.nio.charset.MalformedInputException: Input length = 1异常
        if (Objects.isNull(servletRequest) || Optional.ofNullable(servletRequest.getContentType()).orElse(StringUtils.EMPTY).startsWith("multipart/")) {
            filterChain.doFilter(servletRequest, servletResponse);
            return;
        }

        ServletRequest requestWrapper = null;
        if (servletRequest instanceof HttpServletRequest) {
            requestWrapper = new RequestWrapper((HttpServletRequest) servletRequest);
        }
        if (requestWrapper == null) {
            filterChain.doFilter(servletRequest, servletResponse);
        } else {
            filterChain.doFilter(requestWrapper, servletResponse);
        }
    }

    @Override
    public void destroy() {
    }
}
```







## 兼容LogBack

### 添加依赖

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>2.5.0</version>
</dependency>
<dependency>
    <groupId>com.aliyun.openservices</groupId>
    <artifactId>aliyun-log-logback-appender</artifactId>
    <version>0.1.18</version>
</dependency>
```

[aliyun-log-logback-appender ](https://github.com/aliyun/aliyun-log-logback-appender)



### 分环境配置 logback.xml

> SpringApplication.yml

```yml
# 日志配置
logging:
  level:
    com.qybh: debug
    org.springframework: warn
  # 分环境启用对应配置
  config: classpath:logback-${spring.profiles.active}.xml
```



> logback-dev.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 配置SLS -->
    <appender name="aliyun" class="com.aliyun.openservices.log.logback.LoghubAppender">
        <!-- Required parameters -->
        <!-- Configure account and network  -->
        <endpoint></endpoint>
        <accessKeyId></accessKeyId>
        <accessKeySecret></accessKeySecret>
        <!-- Configure sls -->
        <project>service-hangzhou</project>
        <logStore>test</logStore>
        <!-- Required parameters(end) -->
        <!-- Optional parameters -->
        <!-- 换成自己的 -->
<!--        <topic>TEST</topic>-->
<!--        <source>TEST</source>-->

        <!-- Optional parameters -->
        <totalSizeInBytes>104857600</totalSizeInBytes>
        <maxBlockMs>0</maxBlockMs>
        <ioThreadCount>8</ioThreadCount>
        <batchSizeThresholdInBytes>524288</batchSizeThresholdInBytes>
        <batchCountThreshold>4096</batchCountThreshold>
        <lingerMs>2000</lingerMs>
        <retries>10</retries>
        <baseRetryBackoffMs>100</baseRetryBackoffMs>
        <maxRetryBackoffMs>50000</maxRetryBackoffMs>

        <!-- Optional parameters -->
        <encoder>
            <pattern>%d %-5level [%thread] %logger{0}: %msg</pattern>
        </encoder>

        <!--  Optional parameters -->
        <timeFormat>yyyy-MM-dd'T'HH:mmZ</timeFormat>
        <!--  Optional parameters -->
        <timeZone>UTC</timeZone>

        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <!-- 过滤的级别 -->
            <level>INFO</level>
        </filter>
    </appender>

    <!-- 配置SLS日志响应级别 -->
	<root level="info">
		<appender-ref ref="console" />
        <appender-ref ref="aliyun"/>
	</root>

</configuration> 
```

> logback-prod.xml

......