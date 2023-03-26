---
title: MySQL——子查询优化
date: 2023-03-24 21:30:20
top_img: url(/img/imgs/MySQL.png)
tags: [MySQL]
categories: 面试准备
---

### 子查询语法注意事项

- 子查询必须用小括号扩起来。

    不扩起来的子查询是非法的，比如这样：

  ```sql
  mysql> SELECT SELECT m1 FROM t1;
  
  ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'SELECT m1 FROM t1' at line 1
  ```

- 在`SELECT`子句中的子查询必须是标量子查询。

    如果子查询结果集中有多个列或者多个行，都不允许放在`SELECT`子句中，也就是查询列表中，比如这样就是非法的：

  ```sql
  mysql> SELECT (SELECT m1, n1 FROM t1);
  
  ERROR 1241 (21000): Operand should contain 1 column(s)
  ```

- 在想要得到标量子查询或者行子查询，但又不能保证子查询的结果集只有一条记录时，应该使用`LIMIT 1`语句来限制记录数量。

- 对于`[NOT] IN/ANY/SOME/ALL`子查询来说，子查询中不允许有`LIMIT`语句。

    比如这样是非法的：

  ```sql
  mysql> SELECT * FROM t1 WHERE m1 IN (SELECT * FROM t2 LIMIT 2);
  
  ERROR 1235 (42000): This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'
  ```

    为什么不合法？人家就这么规定的，不解释～ 可能以后的版本会支持吧。正因为`[NOT] IN/ANY/SOME/ALL`子查询不支持`LIMIT`语句，所以子查询中的这些语句也就是多余的了：

  - `ORDER BY`子句

      子查询的结果其实就相当于一个集合，集合里的值排不排序一点儿都不重要，比如下面这个语句中的`ORDER BY`子句简直就是画蛇添足：

    ```sql
    SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 ORDER BY m2);
    ```

  - `DISTINCT`语句

      集合里的值去不去重也没什么意义，比如这样：

    ```sql
    SELECT * FROM t1 WHERE m1 IN (SELECT DISTINCT m2 FROM t2);
    ```

  - 没有聚集函数以及`HAVING`子句的`GROUP BY`子句。

      在没有聚集函数以及`HAVING`子句时，`GROUP BY`子句就是个摆设，比如这样：

    ```sql
    SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 GROUP BY m2);
    ```

      对于这些冗余的语句，查询优化器在一开始就把它们给干掉了。

- 不允许在一条语句中增删改某个表的记录时同时还对该表进行子查询。

    比方说这样：

  ```sql
  mysql> DELETE FROM t1 WHERE m1 < (SELECT MAX(m1) FROM t1);
  
  ERROR 1093 (HY000): You can't specify target table 't1' for update in FROM clause
  ```

## 标量子查询、行子查询的执行方式

  我们经常在下面两个场景中使用到标量子查询或者行子查询：

- `SELECT`子句中，我们前面说过的在查询列表中的子查询必须是标量子查询。
- 子查询使用`=`、`>`、`<`、`>=`、`<=`、`<>`、`!=`、`<=>`等操作符和某个操作数组成一个布尔表达式，这样的子查询必须是标量子查询或者行子查询。

  对于上述两种场景中的不相关标量子查询或者行子查询来说，它们的执行方式是简单的，比方说下面这个查询语句：

```sql
SELECT * FROM s1 
    WHERE key1 = (SELECT common_field FROM s2 WHERE key3 = 'a' LIMIT 1);
```

- 先单独执行`(SELECT common_field FROM s2 WHERE key3 = 'a' LIMIT 1)`这个子查询。
- 然后在将上一步子查询得到的结果当作外层查询的参数再执行外层查询`SELECT * FROM s1 WHERE key1 = ...`。

  也就是说，对于包含不相关的标量子查询或者行子查询的查询语句来说，MySQL会分别独立的执行外层查询和子查询，就当作两个单表查询就好了。

  对于相关的标量子查询或者行子查询来说，比如下面这个查询：

```sql
SELECT * FROM s1 WHERE 
    key1 = (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3 LIMIT 1);
```

- 先从外层查询中获取一条记录，本例中也就是先从`s1`表中获取一条记录。
- 然后从上一步骤中获取的那条记录中找出子查询中涉及到的值，本例中就是从`s1`表中获取的那条记录中找出`s1.key3`列的值，然后执行子查询。
- 最后根据子查询的查询结果来检测外层查询`WHERE`子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则就丢弃。
- 再次执行第一步，获取第二条外层查询中的记录，依次类推～

  也就是说对于一开始介绍的两种使用标量子查询以及行子查询的场景中，`MySQL`优化器的执行方式并没有什么新鲜的。

## IN子查询优化

如果`IN`子查询符合转换为`semi-join`的条件，查询优化器会优先把该子查询为`semi-join`，然后再考虑下面5种执行半连接的策略中哪个成本最低，选择成本最低的那种执行策略来执行子查询。

* Table pullout

- DuplicateWeedout
- LooseScan
- Materialization
- FirstMatch

如果`IN`子查询不符合转换为`semi-join`的条件，那么查询优化器会从下面两种策略中找出一种成本更低的方式执行子查询：

- 先将子查询物化之后再执行查询
- 执行`IN to EXISTS`转换。

### 物化表转连接

将子查询结果集中的记录保存到临时表的过程称之为`物化`（英文名：`Materialize`）。为了方便起见，我们就把那个存储子查询结果集的临时表称之为`物化表`。正因为物化表中的记录都建立了索引（基于内存的物化表有哈希索引，基于磁盘的有B+树索引），通过索引执行`IN`语句判断某个操作数在不在子查询结果集中变得非常快，从而提升了子查询语句的性能。

查询语句转化成表和物化表内连接之后，查询优化器可以评估不同连接顺序需要的成本是多少，选取成本最低的那种查询方式执行查询。

### 将子查询转换为semi-join

#### Table pullout （子查询中的表上拉）

当子查询的查询列表处只有主键或者唯一索引列时，可以直接把子查询中的表`上拉`到外层查询的`FROM`子句中，并把子查询中的搜索条件合并到外层查询的搜索条件中

#### DuplicateWeedout execution strategy （重复值消除）

使用临时表消除`semi-join`结果集中的重复值的方式称之为`DuplicateWeedout`

#### LooseScan execution strategy （松散索引扫描）

```sql
SELECT * FROM s1 
    WHERE key3 IN (SELECT key1 FROM s2 WHERE key1 > 'a' AND key1 < 'b');
```

在子查询中，对于`s2`表的访问可以使用到`key1`列的索引，而恰好子查询的查询列表处就是`key1`列，这样在将该查询转换为半连接查询后，如果将`s2`作为驱动表执行查询的话，那么执行过程就是这样：

![](MySQL——子查询优化/14-03.png)

如图所示，在`s2`表的`idx_key1`索引中，值为`'aa'`的二级索引记录一共有3条，那么只需要取第一条的值到`s1`表中查找`s1.key3 = 'aa'`的记录，如果能在`s1`表中找到对应的记录，那么就把对应的记录加入到结果集。依此类推，其他值相同的二级索引记录，也只需要取第一条记录的值到`s1`表中找匹配的记录，这种虽然是扫描索引，但只取值相同的记录的第一条去做匹配操作的方式称之为`松散索引扫描`。

#### Semi-join Materialization execution strategy

  我们之前介绍的先把外层查询的`IN`子句中的不相关子查询进行物化，然后再进行外层查询的表和物化表的连接本质上也算是一种`semi-join`，只不过由于物化表中没有重复的记录，所以可以直接将子查询转为连接查询。

#### FirstMatch execution strategy （首次匹配）

  `FirstMatch`是一种最原始的半连接执行方式，跟我们年少时认为的相关子查询的执行方式是一样一样的，就是说先取一条外层查询的中的记录，然后到子查询的表中寻找符合匹配条件的记录，如果能找到一条，则将该外层查询的记录放入最终的结果集并且停止查找更多匹配的记录，如果找不到则把该外层查询的记录丢弃掉；然后再开始取下一条外层查询中的记录，重复上面这个过程。

  对于某些使用`IN`语句的相关子查询，比方这个查询：

```sql
SELECT * FROM s1 
  WHERE key1 IN (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3);
```

  它也可以很方便的转为半连接，转换后的语句类似这样：

```sql
SELECT s1.* FROM s1 SEMI JOIN s2 
    ON s1.key1 = s2.common_field AND s1.key3 = s2.key3;
```

  然后就可以使用我们上面介绍过的`DuplicateWeedout`、`LooseScan`、`FirstMatch`等半连接执行策略来执行查询，当然，如果子查询的查询列表处只有主键或者唯一二级索引列，还可以直接使用`table pullout`的策略来执行查询，但是需要大家注意的是，**由于相关子查询并不是一个独立的查询，所以不能转换为物化表来执行查询**。

### 汇总：

#### semi-join的适用条件

  当然，并不是所有包含`IN`子查询的查询语句都可以转换为`semi-join`，只有形如这样的查询才可以被转换为`semi-join`：

```sql
SELECT ... FROM outer_tables 
    WHERE expr IN (SELECT ... FROM inner_tables ...) AND ...
```

  或者这样的形式也可以：

```sql
SELECT ... FROM outer_tables 
    WHERE (oe1, oe2, ...) IN (SELECT ie1, ie2, ... FROM inner_tables ...) AND ...
```

  用文字总结一下，只有符合下面这些条件的子查询才可以被转换为`semi-join`：

- 该子查询必须是和`IN`语句组成的布尔表达式，并且在外层查询的`WHERE`或者`ON`子句中出现。
- 外层查询也可以有其他的搜索条件，只不过和`IN`子查询的搜索条件必须使用`AND`连接起来。
- 该子查询必须是一个单一的查询，不能是由若干查询由`UNION`连接起来的形式。
- 该子查询不能包含`GROUP BY`或者`HAVING`语句或者聚集函数。
- ... 还有一些条件比较少见，就不介绍啦～

#### 不适用于semi-join的情况

  对于一些不能将子查询转位`semi-join`的情况，典型的比如下面这几种：

- 外层查询的WHERE条件中有其他搜索条件与IN子查询组成的布尔表达式使用`OR`连接起来

  ```sql
  SELECT * FROM s1 
      WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a')
          OR key2 > 100;
  ```

- 使用`NOT IN`而不是`IN`的情况

  ```sql
  SELECT * FROM s1 
      WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a');
  ```

- 在`SELECT`子句中的IN子查询的情况

  ```sql
  SELECT key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') FROM s1 ;
  ```

- 子查询中包含`GROUP BY`、`HAVING`或者聚集函数的情况

  ```sql
  SELECT * FROM s1 
      WHERE key2 IN (SELECT COUNT(*) FROM s2 GROUP BY key1);
  ```

- 子查询中包含`UNION`的情况

  ```sql
  SELECT * FROM s1 WHERE key1 IN (
      SELECT common_field FROM s2 WHERE key3 = 'a' 
      UNION
      SELECT common_field FROM s2 WHERE key3 = 'b'
  );
  ```

  `MySQL`仍然留了两手绝活来优化不能转为`semi-join`查询的子查询，那就是：

- 对于不相关子查询来说，可以尝试把它们物化之后再参与查询

    比如我们上面提到的这个查询：

  ```sql
  SELECT * FROM s1 
      WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a');
  ```

    先将子查询物化，然后再判断`key1`是否在物化表的结果集中可以加快查询执行的速度。

  > 小贴士：请注意这里将子查询物化之后不能转为和外层查询的表的连接，只能是先扫描s1表，然后对s1表的某条记录来说，判断该记录的key1值在不在物化表中。

- 不管子查询是相关的还是不相关的，都可以把`IN`子查询尝试专为`EXISTS`子查询

    其实对于任意一个IN子查询来说，都可以被转为`EXISTS`子查询，通用的例子如下：

  ```sql
  outer_expr IN (SELECT inner_expr FROM ... WHERE subquery_where);
  ```

    可以被转换为：

  ```sql
  EXISTS (SELECT inner_expr FROM ... WHERE subquery_where AND outer_expr=inner_expr);
  ```

    当然这个过程中有一些特殊情况，比如在`outer_expr`或者`inner_expr`值为`NULL`的情况下就比较特殊。因为有`NULL`值作为操作数的表达式结果往往是`NULL`，比方说：

  ```sql
  mysql> SELECT NULL IN (1, 2, 3);
  +-------------------+
  | NULL IN (1, 2, 3) |
  +-------------------+
  |              NULL |
  +-------------------+
  1 row in set (0.00 sec)
  
  mysql> SELECT 1 IN (1, 2, 3);
  +----------------+
  | 1 IN (1, 2, 3) |
  +----------------+
  |              1 |
  +----------------+
  1 row in set (0.00 sec)
  
  mysql> SELECT NULL IN (NULL);
  +----------------+
  | NULL IN (NULL) |
  +----------------+
  |           NULL |
  +----------------+
  1 row in set (0.00 sec)
  ```

    而`EXISTS`子查询的结果肯定是`TRUE`或者`FASLE`：

  ```sql
  mysql> SELECT EXISTS (SELECT 1 FROM s1 WHERE NULL = 1);
  +------------------------------------------+
  | EXISTS (SELECT 1 FROM s1 WHERE NULL = 1) |
  +------------------------------------------+
  |                                        0 |
  +------------------------------------------+
  1 row in set (0.01 sec)
  
  mysql> SELECT EXISTS (SELECT 1 FROM s1 WHERE 1 = NULL);
  +------------------------------------------+
  | EXISTS (SELECT 1 FROM s1 WHERE 1 = NULL) |
  +------------------------------------------+
  |                                        0 |
  +------------------------------------------+
  1 row in set (0.00 sec)
  
  mysql> SELECT EXISTS (SELECT 1 FROM s1 WHERE NULL = NULL);
  +---------------------------------------------+
  | EXISTS (SELECT 1 FROM s1 WHERE NULL = NULL) |
  +---------------------------------------------+
  |                                           0 |
  +---------------------------------------------+
  1 row in set (0.00 sec)
  ```

    但是幸运的是，我们大部分使用`IN`子查询的场景是把它放在`WHERE`或者`ON`子句中，而`WHERE`或者`ON`子句是不区分`NULL`和`FALSE`的，比方说：

  ```sql
  mysql> SELECT 1 FROM s1 WHERE NULL;
  Empty set (0.00 sec)
  
  mysql> SELECT 1 FROM s1 WHERE FALSE;
  Empty set (0.00 sec)
  ```

    所以只要我们的`IN`子查询是放在`WHERE`或者`ON`子句中的，那么`IN -> EXISTS`的转换就是没问题的。说了这么多，为什么要转换呢？这是因为不转换的话可能用不到索引，比方说下面这个查询：

  ```sql
  SELECT * FROM s1
      WHERE key1 IN (SELECT key3 FROM s2 where s1.common_field = s2.common_field) 
          OR key2 > 1000;
  ```

    这个查询中的子查询是一个相关子查询，而且子查询执行的时候不能使用到索引，但是将它转为`EXISTS`子查询后却可以使用到索引：

  ```sql
  SELECT * FROM s1
      WHERE EXISTS (SELECT 1 FROM s2 where s1.common_field = s2.common_field AND s2.key3 = s1.key1) 
          OR key2 > 1000;
  ```

    转为`EXISTS`子查询时便可以使用到`s2`表的`idx_key3`索引了。

    需要注意的是，如果`IN`子查询不满足转换为`semi-join`的条件，又不能转换为物化表或者转换为物化表的成本太大，那么它就会被转换为`EXISTS`查询。

> 小贴士：在MySQL5.5以及之前的版本没有引进semi-join和物化的方式优化子查询时，优化器都会把IN子查询转换为EXISTS子查询，好多同学就惊呼我明明写的是一个不相关子查询，为什么要按照执行相关子查询的方式来执行呢？所以当时好多声音都是建议大家把子查询转为连接，不过随着MySQL的发展，最近的版本中引入了非常多的子查询优化策略，大家可以稍微放心的使用子查询了，内部的转换工作优化器会为大家自动实现。

## ANY/ALL子查询优化

  如果ANY/ALL子查询是不相关子查询的话，它们在很多场合都能转换成我们熟悉的方式去执行，比方说：

| 原始表达式                    | 转换为                         |
| ----------------------------- | ------------------------------ |
| < ANY (SELECT inner_expr ...) | < (SELECT MAX(inner_expr) ...) |
| > ANY (SELECT inner_expr ...) | > (SELECT MIN(inner_expr) ...) |
| < ALL (SELECT inner_expr ...) | < (SELECT MIN(inner_expr) ...) |
| > ALL (SELECT inner_expr ...) | > (SELECT MAX(inner_expr) ...) |

### [NOT\] EXISTS子查询的执行

  如果`[NOT] EXISTS`子查询是不相关子查询，可以先执行子查询，得出该`[NOT] EXISTS`子查询的结果是`TRUE`还是`FALSE`，并重写原先的查询语句，比如对这个查询来说：

```sql
SELECT * FROM s1 
    WHERE EXISTS (SELECT 1 FROM s2 WHERE key1 = 'a') 
        OR key2 > 100;
```

  因为这个语句里的子查询是不相关子查询，所以优化器会首先执行该子查询，假设该EXISTS子查询的结果为`TRUE`，那么接着优化器会重写查询为：

```sql
SELECT * FROM s1 
    WHERE TRUE OR key2 > 100;
```

  进一步简化后就变成了：

```sql
SELECT * FROM s1 
    WHERE TRUE;
```

  对于相关的`[NOT] EXISTS`子查询来说，比如这个查询：

```sql
SELECT * FROM s1 
    WHERE EXISTS (SELECT 1 FROM s2 WHERE s1.common_field = s2.common_field);
```

  很不幸，这个查询只能按照我们年少时的那种执行相关子查询的方式来执行。不过如果`[NOT] EXISTS`子查询中如果可以使用索引的话，那查询速度也会加快不少，比如：

```sql
SELECT * FROM s1 
    WHERE EXISTS (SELECT 1 FROM s2 WHERE s1.common_field = s2.key1);
```

  上面这个`EXISTS`子查询中可以使用`idx_key1`来加快查询速度。

### 对于派生表的优化

  我们前面说过把子查询放在外层查询的`FROM`子句后，那么这个子查询的结果相当于一个`派生表`，比如下面这个查询：

```sql
SELECT * FROM  (
        SELECT id AS d_id,  key3 AS d_key3 FROM s2 WHERE key1 = 'a'
    ) AS derived_s1 WHERE d_key3 = 'a';
```

  子查询`( SELECT id AS d_id,  key3 AS d_key3 FROM s2 WHERE key1 = 'a')`的结果就相当于一个派生表，这个表的名称是`derived_s1`，该表有两个列，分别是`d_id`和`d_key3`。

  对于含有`派生表`的查询，`MySQL`提供了两种执行策略：

- 最容易想到的就是把派生表物化。

    我们可以将派生表的结果集写到一个内部的临时表中，然后就把这个物化表当作普通表一样参与查询。当然，在对派生表进行物化时，设计`MySQL`的大佬使用了一种称为`延迟物化`的策略，也就是在查询中真正使用到派生表时才回去尝试物化派生表，而不是还没开始执行查询呢就把派生表物化掉。比方说对于下面这个含有派生表的查询来说：

  ```sql
  SELECT * FROM (
          SELECT * FROM s1 WHERE key1 = 'a'
      ) AS derived_s1 INNER JOIN s2
      ON derived_s1.key1 = s2.key1
      WHERE s2.key2 = 1;
  ```

    如果采用物化派生表的方式来执行这个查询的话，那么执行时首先会到`s1`表中找出满足`s1.key2 = 1`的记录，如果压根儿找不到，说明参与连接的`s1`表记录就是空的，所以整个查询的结果集就是空的，所以也就没有必要去物化查询中的派生表了。

- 将派生表和外层的表合并，也就是将查询重写为没有派生表的形式

    我们来看这个贼简单的包含派生表的查询：

  ```sql
  SELECT * FROM (SELECT * FROM s1 WHERE key1 = 'a') AS derived_s1;
  ```

    这个查询本质上就是想查看`s1`表中满足`key1 = 'a'`条件的的全部记录，所以和下面这个语句是等价的：

  ```sql
  SELECT * FROM s1 WHERE key1 = 'a';
  ```

    对于一些稍微复杂的包含派生表的语句，比如我们上面提到的那个：

  ```sql
  SELECT * FROM (
          SELECT * FROM s1 WHERE key1 = 'a'
      ) AS derived_s1 INNER JOIN s2
      ON derived_s1.key1 = s2.key1
      WHERE s2.key2 = 1;
  ```

    我们可以将派生表与外层查询的表合并，然后将派生表中的搜索条件放到外层查询的搜索条件中，就像这样：

  ```sql
  SELECT * FROM s1 INNER JOIN s2 
      ON s1.key1 = s2.key1
      WHERE s1.key1 = 'a' AND s2.key2 = 1;
  ```

    这样通过将外层查询和派生表合并的方式成功的消除了派生表，也就意味着我们没必要再付出创建和访问临时表的成本了。可是并不是所有带有派生表的查询都能被成功的和外层查询合并，当派生表中有这些语句就不可以和外层查询合并：

  - 聚集函数，比如MAX()、MIN()、SUM()什么的
  - DISTINCT
  - GROUP BY
  - HAVING
  - LIMIT
  - UNION 或者 UNION ALL
  - 派生表对应的子查询的`SELECT`子句中含有另一个子查询
  - ... 还有些不常用的情况就不多说了～

  所以`MySQL`在执行带有派生表的时候，优先尝试把派生表和外层查询合并掉，如果不行的话，再把派生表物化掉执行查询。





完整文章为：《MySQL 是怎样运行的：从根儿上理解 MySQL》——第14章 不好看就要多整容-MySQL基于规则的优化（内含关于子查询优化二三事儿）