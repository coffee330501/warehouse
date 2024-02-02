熔断熔断！

[Sentinal 官方文档](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)

# 使用

服务中引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```



# 控制台

云服务器放行8080端口

[下载地址](https://github.com/alibaba/Sentinel/releases)

> 启动 nohup java -jar sentinel-dashboard-1.8.7.jar >nohup.out 2>&1 &

默认端口为8080（可以通过 -Dserver.port=8080 指定端口）

默认账号密码 sentinel/sentinel