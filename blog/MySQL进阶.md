## 自定义排序

**order by field()**

```sql
SELECT * FROM movies ORDER BY FIELD(movie_name,'神话','猎场','芳华','花木兰')
```



## 空置排序

默认空置(null)数据会在最前面，可以使用以下语法

```sql
-- actors为排序字段，为空时为1，不为空为0（0在前面，1在后面，大的在后面）
SELECT * FROM movie ORDER BY if(ISNULL(actors),1,0), actors, price;
```



## 分组连接

```sql
SELECT actors, GROUP_CONCAT(movie_name) FROM movies GROUP BY actors;

SELECT actors, GROUP_CONCAT(movie_name ORDER BY price DESC SEPARATOR '_') FROM movies GROUP BY actors;
```



## 分组后汇总

```sql
-- 会额外生成一条数据在最后来汇总金额
SELECT actors,SUM(price) FROM movies GROUP BY actors WITYH ROLLUP;
```



## 子查询提取

```sql
WITH 
	m1 AS (SELECT * FROM movies where price > 50),
	m2 AS (SELECT * FROM movies where price > 65)
SELECT * FROM m1 WHERE m1.id not in (SELECT m2.id FROM m2) AND m1.actors = '刘亦菲'; 
```



## 优雅处理数据插入、更新时主键、唯一键重复

### IGNORE

重复时自动忽略重复的数据，不影响后面数据的插入

```sql
INSERT IGNORE INTO movies VALUES (...);
```

### REPLACE

插入的记录重复时会先删除表中重复的再插入。

```sql
REPLACE INTO movies VALUES (...)
```



### ON DUPLICATE KEY UPDATE

重复时执行UPDATE

```sql
INSERT INTO movies VALUES (...) ON DUPLICATE KEY UPDATE price = price + 10;
```

