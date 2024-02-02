参考:

[Spring Cloud Gateway 官方文档](https://spring.io/projects/spring-cloud-gateway)

# 依赖

## 父模块依赖

```xml
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-cloud.version>2021.0.5</spring-cloud.version>
    <spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>
    <spring-boot.version>2.7.7</spring-boot.version>
</properties>

<dependencies>
    <!-- bootstrap 启动器 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <!-- spring-cloud-dependencies -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <!-- spring-cloud-alibaba-dependencies -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <!-- SpringBoot 依赖配置 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

需要注意的是，`bootstrap 启动器`是直接引入的，因为远程加载Nacos配置需要使用bootstrap配置文件启动，每个服务基本都要这个依赖。



## Gateway依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.github.coffee330501</groupId>
        <artifactId>MicroserviceLearn</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>gateway</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <!-- 不配置的话无法发现服务 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>2.3.0</version>
        </dependency>

        <!-- 服务发现 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
    </dependencies>
</project>
```



# 参数配置

## filters

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: card1
          # lb是请求服务名的协议,也可以使用http
          uri: lb://card
#          uri: https://baidu.com
          # 当用户请求/card/**时，会转到card服务下
          predicates:
            - Path=/card/**
          filters:
            - StripPrefix=1
```

### StripPrefix

去掉一段路径

例如，前端请求网关 `ip:port/card/test` 时，会将 `/card` 去掉，最终请求到具体服务的是 `服务ip:port/test`





# Tips

> 启动报错 java.lang.NoSuchMethodError: org.yaml.snakeyaml.LoaderOptions.setMaxAliasesForCollections(I)V

![image-20240130150245087](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240130150245087.png)

可看出依赖冲突

将低版本的依赖排除掉就行了

> 