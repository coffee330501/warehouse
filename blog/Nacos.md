[官方文档](https://nacos.io/docs/latest/what-is-nacos/)



# 简介

是一个服务发现、配置管理和服务管理平台



# 使用

**启动服务器**

/bin下

```shell
sh startup.sh -m standalone
```

**关闭服务器**

```shell
sh shutdown.sh
```



# Tips

> idea开启微服务控制台

![image-20240130111625306](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240130111625306.png)

> PathVariable要把key写上

```java
@FeignClient("consumption")
public interface ConsumptionClient {
    @RequestMapping(method = RequestMethod.GET,value = "{cardId}")
    String consumption(@PathVariable("cardId") String cardId);
}
```

