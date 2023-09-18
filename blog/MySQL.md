## 使用@声明用户变量

**参考:** [MySQL 中的系统变量和用户变量](https://blog.csdn.net/qq_66862911/article/details/130894917)



用户变量赋值有两种方式: 一种是直接用"=“号，另一种是用”:=“号。

其区别在于:

- 使用set命令对用户变量进行赋值时，两种方式都可以使用；
- 用select语句时，只能用”:=“方式，因为select语句中，”="号被看作是比较操作符。

**@变量 := 值**

给变量赋一个初始值

```sql
SELECT @a := 2 as a
```

![image-20230918155345658](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20230918155345658.png)

```sql
SELECT
	id,
	@t := @t + 1 
FROM
	( SELECT a.t, b.id FROM bd_supplier b CROSS JOIN ( SELECT @t := 0 AS t ) a ) c;
```

![image-20230918162257293](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20230918162257293.png)

**默认声明的是会话级别的变量**