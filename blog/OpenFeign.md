参考:

[openfeign官方文档](https://cloud.spring.io/spring-cloud-openfeign/reference/html/)

[Nacos集成OpenFeign](https://blog.csdn.net/qq825478739/article/details/122095117)



# 依赖

```xml
<!-- SpringCloud openfeign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- SpringCloud Loadbalancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```



# 使用

## 开启openfeign

配置 `@EnableFeignClients`

```java
@SpringBootApplication
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```



## 声明远程服务接口

```java
@FeignClient("consumption")
public interface ConsumptionClient {
    @GetMapping("/consumption/{cardId}")
    String consumption(@PathVariable("cardId") String cardId);

    @GetMapping("/consumption/post")
    String post(@SpringQueryMap ConsumptionRecord consumptionRecord);
}
```

**FeignClient**： 声明接口为远程调用接口，并且调用的服务名为`consumption`。也可以指定url，但是指定了url后不回进行服务的负载均衡。

**注意点：**

- 不能在接口上加`@RequestMapping`；
- `@PathVariable`一定要指定明确的参数名，即使只有一个路径参数。
- 默认的 OpenFeign QueryMap 注释与 Spring 不兼容，因为它缺少属性`value`。所以当 POJO 或 Map 参数注解为查询参数时，需要使用openfeign提供的注解`@SpringQueryMap`，如上示例；

其他的像正常controller接口一样。



**@FeignClient 源码**

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface FeignClient {
    @AliasFor("name")
    String value() default "";

    String contextId() default "";

    @AliasFor("value")
    String name() default "";

    /** @deprecated */
    @Deprecated
    String qualifier() default "";

    String[] qualifiers() default {};

    String url() default "";

    boolean decode404() default false;

    Class<?>[] configuration() default {};

    Class<?> fallback() default void.class;

    Class<?> fallbackFactory() default void.class;

    String path() default "";

    boolean primary() default true;
}
```

- **value**：name的别名，指定服务名;

- **contextId**：当我们有多个相同name、value不同配置的feign客户端时，需要指定`contextId属性`以避免bean名称冲突；

  ```java
  @FeignClient(contextId = "fooClient", name = "stores", configuration = FooConfiguration.class)
  public interface FooClient {
      //..
  }
  ```

  ```java
  @FeignClient(contextId = "barClient", name = "stores", configuration = BarConfiguration.class)
  public interface BarClient {
      //..
  }
  ```

- **url**：指定请求url



> Tips

`name` 和属性支持占位符 `url`

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {
    //..
}
```



## 日志

`application.properties`

```properties
logging.level.project.user.UserClient: DEBUG
```



# 源码解析

测试请求是有负载均衡的

## 入口 ReflectiveFeign#invoke

```java
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (!"equals".equals(method.getName())) {
                if ("hashCode".equals(method.getName())) {
                    return this.hashCode();
                } else {
                    return "toString".equals(method.getName()) ? this.toString() : ((InvocationHandlerFactory.MethodHandler)this.dispatch.get(method)).invoke(args);
                }
            } else {
                try {
                    Object otherHandler = args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
                    return this.equals(otherHandler);
                } catch (IllegalArgumentException var5) {
                    return false;
                }
            }
        }
```

进入 `this.dispatch.get(method))`

![image-20240131163046817](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131163046817.png)

dispatch是一个map，key为方法，value为方法的处理器。这里通过调用的方法名找到了对应的方法处理器。

找到方法处理器后通过invoke()方法调用。



## 进入 SynchronousMethodHandler#invoke

```java
    public Object invoke(Object[] argv) throws Throwable {
        RequestTemplate template = this.buildTemplateFromArgs.create(argv);
        Request.Options options = this.findOptions(argv);
        Retryer retryer = this.retryer.clone();

        while(true) {
            try {
                return this.executeAndDecode(template, options);
            } catch (RetryableException var9) {
                RetryableException e = var9;

                try {
                    retryer.continueOrPropagate(e);
                } catch (RetryableException var8) {
                    Throwable cause = var8.getCause();
                    if (this.propagationPolicy == ExceptionPropagationPolicy.UNWRAP && cause != null) {
                        throw cause;
                    }

                    throw var8;
                }

                if (this.logLevel != Level.NONE) {
                    this.logger.logRetry(this.metadata.configKey(), this.logLevel);
                }
            }
        }
    }
```

首先，创建了一个RequestTemplate请求模板

![image-20240131165254327](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131165254327.png)

然后创建options，存放连接时间、超时时间等配置

![image-20240131165816818](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131165816818.png)

然后clone了一个失败重试的策略类(retryer)。

最后执行 `executeAndDecode` 方法

```java
   Object executeAndDecode(RequestTemplate template, Request.Options options) throws Throwable {
        Request request = this.targetRequest(template);
        if (this.logLevel != Level.NONE) {
            this.logger.logRequest(this.metadata.configKey(), this.logLevel, request);
        }

        long start = System.nanoTime();

        Response response;
        try {
            response = this.client.execute(request, options);
            response = response.toBuilder().request(request).requestTemplate(template).build();
        } catch (IOException var13) {
            if (this.logLevel != Level.NONE) {
                this.logger.logIOException(this.metadata.configKey(), this.logLevel, var13, this.elapsedTime(start));
            }

            throw FeignException.errorExecuting(request, var13);
        }

        long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
        if (this.decoder != null) {
            return this.responseInterceptor.aroundDecode(new InvocationContext(this.decoder, this.metadata.returnType(), response));
        } else {
            CompletableFuture<Object> resultFuture = new CompletableFuture();
            this.asyncResponseHandler.handleResponse(resultFuture, this.metadata.configKey(), response, this.metadata.returnType(), elapsedTime);

            try {
                if (!resultFuture.isDone()) {
                    throw new IllegalStateException("Response handling not done");
                } else {
                    return resultFuture.join();
                }
            } catch (CompletionException var12) {
                Throwable cause = var12.getCause();
                if (cause != null) {
                    throw cause;
                } else {
                    throw var12;
                }
            }
        }
    }

```

先通过`targetRequest`获取了请求，再执行

## 调用请求

```java
    public Response execute(Request request, Request.Options options) throws IOException {
        URI originalUri = URI.create(request.url());
      	// 获取服务名
        String serviceId = originalUri.getHost();
        Assert.state(serviceId != null, "Request URI does not contain a valid hostname: " + originalUri);
        LoadBalancedRetryPolicy retryPolicy = this.loadBalancedRetryFactory.createRetryPolicy(serviceId, this.loadBalancerClient);
        RetryTemplate retryTemplate = this.buildRetryTemplate(serviceId, request, retryPolicy);
        return (Response)retryTemplate.execute((context) -> {
            Request feignRequest = null;
            ServiceInstance retrievedServiceInstance = null;
            Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator.getSupportedLifecycleProcessors(this.loadBalancerClientFactory.getInstances(serviceId, LoadBalancerLifecycle.class), RetryableRequestContext.class, ResponseData.class, ServiceInstance.class);
            String hint = this.getHint(serviceId);
            DefaultRequest<RetryableRequestContext> lbRequest = new DefaultRequest(new RetryableRequestContext((ServiceInstance)null, LoadBalancerUtils.buildRequestData(request), hint));
            if (context instanceof LoadBalancedRetryContext) {
                LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext)context;
                ServiceInstance serviceInstance = lbContext.getServiceInstance();
                if (serviceInstance == null) {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Service instance retrieved from LoadBalancedRetryContext: was null. Reattempting service instance selection");
                    }

                    ServiceInstance previousServiceInstance = lbContext.getPreviousServiceInstance();
                    ((RetryableRequestContext)lbRequest.getContext()).setPreviousServiceInstance(previousServiceInstance);
                    supportedLifecycleProcessors.forEach((lifecycle) -> {
                        lifecycle.onStart(lbRequest);
                    });
                    // 
                    retrievedServiceInstance = this.loadBalancerClient.choose(serviceId, lbRequest);
                    if (LOG.isDebugEnabled()) {
                        LOG.debug(String.format("Selected service instance: %s", retrievedServiceInstance));
                    }

                    lbContext.setServiceInstance(retrievedServiceInstance);
                }

                if (retrievedServiceInstance == null) {
                    if (LOG.isWarnEnabled()) {
                        LOG.warn("Service instance was not resolved, executing the original request");
                    }

                    org.springframework.cloud.client.loadbalancer.Response<ServiceInstance> lbResponsex = new DefaultResponse(retrievedServiceInstance);
                    supportedLifecycleProcessors.forEach((lifecycle) -> {
                        lifecycle.onComplete(new CompletionContext(Status.DISCARD, lbRequest, lbResponsex));
                    });
                    feignRequest = request;
                } else {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug(String.format("Using service instance from LoadBalancedRetryContext: %s", retrievedServiceInstance));
                    }

                    String reconstructedUrl = this.loadBalancerClient.reconstructURI(retrievedServiceInstance, originalUri).toString();
                    feignRequest = this.buildRequest(request, reconstructedUrl);
                }
            }

            org.springframework.cloud.client.loadbalancer.Response<ServiceInstance> lbResponse = new DefaultResponse(retrievedServiceInstance);
            LoadBalancerProperties loadBalancerProperties = this.loadBalancerClientFactory.getProperties(serviceId);
            Response response = LoadBalancerUtils.executeWithLoadBalancerLifecycleProcessing(this.delegate, options, feignRequest, lbRequest, lbResponse, supportedLifecycleProcessors, retrievedServiceInstance != null, loadBalancerProperties.isUseRawStatusCodeInResponseData());
            int responseStatus = response.status();
            if (retryPolicy != null && retryPolicy.retryableStatusCode(responseStatus)) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug(String.format("Retrying on status code: %d", responseStatus));
                }

                byte[] byteArray = response.body() == null ? new byte[0] : StreamUtils.copyToByteArray(response.body().asInputStream());
                response.close();
                throw new LoadBalancerResponseStatusCodeException(serviceId, response, byteArray, URI.create(request.url()));
            } else {
                return response;
            }
        }, new LoadBalancedRecoveryCallback<Response, Response>() {
            protected Response createResponse(Response response, URI uri) {
                return response;
            }
        });
    }
```



`LoadBalancerLifecycleValidator.getSupportedLifecycleProcessors`  拿到了一个空列表

随后进入choose方法

```java
    public <T> ServiceInstance choose(String serviceId, Request<T> request) {
        ReactiveLoadBalancer<ServiceInstance> loadBalancer = this.loadBalancerClientFactory.getInstance(serviceId);
        if (loadBalancer == null) {
            return null;
        } else {
            Response<ServiceInstance> loadBalancerResponse = (Response)Mono.from(loadBalancer.choose(request)).block();
            return loadBalancerResponse == null ? null : (ServiceInstance)loadBalancerResponse.getServer();
        }
    }
```

先获取了一个 `loadBalancer` ，随后通过 `loadBalancer` 获取了服务真实的调用地址。

![image-20240131172939730](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131172939730.png)

![image-20240131173027242](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131173027242.png)

### choose中的负载均衡策略

![image-20240131173536909](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240131173536909.png)

其中

- `RandomLoadBalancer` 和 `RooundRobinLoadBalancer` 是 spring-cloud-loadbalancer提供的
- `NacosLoadBalancer` 是 Nacos 提供的

这里走了 `RooundRobinLoadBalancer ` 是循环调用服务的客户端