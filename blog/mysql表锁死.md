```sql
-- 查询正在使用的表
show open tables where in_use > 0;
```

![image-20240515102441291](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240515102441291.png)

```sql
-- 当前运行的事务
SELECT * FROM information_schema.INNODB_TRX;
```

![image-20240515102532491](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20240515102532491.png)

```sql
SELECT concat('KILL ',id,';')
FROM information_schema.processlist p
INNER JOIN  information_schema.INNODB_TRX x
ON p.id=x.trx_mysql_thread_id
-- 有时候可以不加
WHERE db='steel_pay';
```

最后执行查询出的语句

kill *** 即可