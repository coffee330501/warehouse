参考：

[SpringCloud Alibaba实战第五课 链路追踪sleuth与skywalking](https://blog.csdn.net/fegus/article/details/124643581)

[Zipkin 官方文档](https://zipkin.io/pages/quickstart.html?fileGuid=xxQTRXtVcqtHK6j8)

[Docker搭建Zipkin，实现数据持久化到MySQL、ES](https://zhuanlan.zhihu.com/p/652677976)



> 前置条件

项目已整合Nacos与Openfeign



# 使用

在每个链路模块中增加依赖

```xml
<!-- Spring Cloud Sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```



sleuth收集信息是有一定的比率的，默认的采样率是0.1，配置此值的方式在配置文件中增加spring.sleuth.sampler.percentage参数配置（如果不配置默认0.1），如果我们调大此值为1，可以看到信息收集就更及时。但是当这样调整后，我们会发现我们的rest接口调用速度比0.1的情况下慢了很多，即使在0.1的采样率下，我们多次刷新consumer的接口，会发现对同一个请求两次耗时信息相差非常大，如果取消spring-cloud-sleuth后我们再测试，会发现并没有这种情况，可以看到这种方式追踪服务调用链路会给我们业务程序性能带来一定的影响。

> sleuth采样率，默认为0.1，值越大收集越及时，但性能影响也越大

```properties
spring.sleuth.sampler.percentage=1
```



# Zipkin可视化

**云服务器开放9411端口**

## 使用Docker安装Zipkin

```shell
docker pull openzipkin/zipkin
docker run --name zipkin -d --restart=always -p 9411:9411 openzipkin/zipkin
```

- `--restart=always` 可以让容器在退出后自动重启,保证可用性
- `-p 9411:9411` 是端口映射,将容器内部默认的 9411 端口映射到宿主机的 9411 端口,方便访问
- 指定镜像版本号 `openzipkin/zipkin:2.21.7` 是个好习惯,避免使用默认 latest 标签导致不可控的问题
- 如果需要调整配置,可以使用 `-e` 参数设置环境变量,例如:`-e JAVA_OPTS="-Xms512m -Xmx512m"` 来控制 Zipkin 的内存
- 数据默认存放在内存中,建议通过 `-v` 参数映射卷持久化数据,避免重启后丢失



访问: IP:9411

![image-20240202131613544](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240202131613544.png)

可以看到什么都没有，因为没有服务注册到Zipkin上



## 将服务注册到Zipkin

父工程引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.2.8.RELEASE</version>
</dependency>
```

添加配置

```properties
spring.zipkin.base-url=http://feng.com:9411/
```

重启服务，调用一次



![image-20240202140313217](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240202140313217.png)

![image-20240202140357048](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240202140357048.png)