# Mybatis拦截器

Mybatis中提供了`org.apache.ibatis.plugin.Interceptor`接口用于在SQL执行前后做一些处理。

## 使用

实现Interceptor

```java
@Component
@Intercepts({@Signature(method = "update", type = Executor.class, args = {MappedStatement.class, Object.class})})
public class MybatisInterceptor implements Interceptor {
    private Logger logger = LoggerFactory.getLogger(MybatisInterceptor.class);
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        try{
            // 被拦截的参数
            Object[] args = invocation.getArgs();
        }catch (Exception e){
            logger.error("MybatisInterceptor: ",e);
        }
        return invocation.proceed();
    }
}
```



**@Intercepts**

Mybatis会拿到拦截器的`@Intercepts`注解，并根据注解的value值来确定拦截的方法。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface Intercepts {
    Signature[] value();
}
```

Intercepts的value类型为Signature[]



**@Signature**

`@Signature`包含了三个参数

- type：要拦截的类
- method：要拦截的方法名
- args：要拦截的方法的参数（因为java有重载，只有方法名无法确定唯一的方法）

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Signature {
    Class<?> type();

    String method();

    Class<?>[] args();
}
```



**Executor.class**

上文中的自定义注解中@Signature为`@Signature(method = "update", type = Executor.class, args = {MappedStatement.class, Object.class})`

是因为Executor类是所有SQL执行类的基类，拦截了它的update方法就拦截了所有非查询语句的执行。

另外，`invocation.getArgs();`获取到的参数就是传入Executor类update方法的参数。

```java
public interface Executor {
    ResultHandler NO_RESULT_HANDLER = null;

    int update(MappedStatement var1, Object var2) throws SQLException;

    <E> List<E> query(MappedStatement var1, Object var2, RowBounds var3, ResultHandler var4, CacheKey var5, BoundSql var6) throws SQLException;

    <E> List<E> query(MappedStatement var1, Object var2, RowBounds var3, ResultHandler var4) throws SQLException;

    <E> Cursor<E> queryCursor(MappedStatement var1, Object var2, RowBounds var3) throws SQLException;

    List<BatchResult> flushStatements() throws SQLException;

    void commit(boolean var1) throws SQLException;

    void rollback(boolean var1) throws SQLException;

    CacheKey createCacheKey(MappedStatement var1, Object var2, RowBounds var3, BoundSql var4);

    boolean isCached(MappedStatement var1, CacheKey var2);

    void clearLocalCache();

    void deferLoad(MappedStatement var1, MetaObject var2, String var3, CacheKey var4, Class<?> var5);

    Transaction getTransaction();

    void close(boolean var1);

    boolean isClosed();

    void setExecutorWrapper(Executor var1);
}
```



**最后**

在SqlSessionFactory的配置中注册拦截器

```java
    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception
    {
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        // ...
        // 注册拦截器
        sessionFactory.setPlugins(mybatisInterceptor);
        return sessionFactory.getObject();
    }
```



> 注意

- 报错`No @Intercepts annotation was found in interceptor`，检查自定义的拦截器上有没有添加`@Component`