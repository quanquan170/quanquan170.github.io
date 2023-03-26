---
title: MySQL——访问方法合集
date: 2023-03-21 11:59:20
top_img: url(/img/imgs/MySQL.png)
tags: [MySQL]
categories: 面试准备
---

执行查询语句的方式称之为`访问方法`或者`访问类型`。可以通过explain语句在执行计划的`type`列查看。

完整的访问方法如下：`system`，`const`，`eq_ref`，`ref`，`fulltext`，`ref_or_null`，`index_merge`，`unique_subquery`，`index_subquery`，`range`，`index`，`ALL`。

除了`All`这个访问方法外，其余的访问方法都能用到索引，除了`index_merge`访问方法外，其余的访问方法都最多只能用到一个索引

建表语句：

```sql
CREATE TABLE single_table (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

- 为`id`列建立的聚簇索引。
- 为`key1`列建立的`idx_key1`二级索引。
- 为`key2`列建立的`idx_key2`二级索引，而且该索引是唯一二级索引。
- 为`key3`列建立的`idx_key3`二级索引。
- 为`key_part1`、`key_part2`、`key_part3`列建立的`idx_key_part`二级索引，这也是一个联合索引。

## 单表访问方法（InnoDB）

### const

通过**主键**或者**唯一二级索引列与常数的等值比较**来定位**一条记录**

### ref

搜索条件为**二级索引列与常数等值比较**，采用二级索引来执行查询的访问方法

注：

- 二级索引列值为`NULL`的情况

    不论是普通的二级索引，还是唯一二级索引，它们的索引列对包含`NULL`值的数量并不限制，所以我们采用`key IS NULL`这种形式的搜索条件最多只能使用`ref`的访问方法，而不是`const`的访问方法。

- 对于某个包含多个索引列的二级索引来说，只要是最**左边的连续索引列是与常数的等值比较**就**可能**采用`ref`的访问方法（还可能是全表扫描）

### ref_or_null

不仅想找出某个二级索引列的值等于某个常数的记录，还想把该列的值为`NULL`的记录也找出来，当使用**二级索引**而不是全表扫描的方式执行该查询时，这种类型的查询使用的访问方法就称为`ref_or_null`

### range

```sql
SELECT * 
FROM single_table 
WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);
```

从数学的角度看，每一个所谓的范围都是数轴上的一个`区间`，3个范围也就对应着3个区间：

- 范围1：`key2 = 1438`
- 范围2：`key2 = 6328`
- 范围3：`key2 ∈ [38, 79]`，注意这里是闭区间。

  我们可以把那种索引列等值匹配的情况称之为`单点区间`，上面所说的`范围1`和`范围2`都可以被称为单点区间，像`范围3`这种的我们可以称为连续范围区间。

​		如果采用`二级索引 + 回表`的方式来执行的话，那么此时的搜索条件就不只是要求索引列与常数的等值匹配了，而是索引列需要匹配某个或某些范围的值

### index

```sql
SELECT key_part1, key_part2, key_part3 
FROM single_table 
WHERE key_part2 = 'abc';
```

 由于`key_part2`并不是联合索引`idx_key_part`最左索引列，所以我们无法使用`ref`或者`range`访问方法来执行这个语句。但是这个查询符合下面这两个条件：

- 它的查询列表只有3个列：`key_part1`, `key_part2`, `key_part3`，而索引`idx_key_part`又包含这三个列。
- 搜索条件中只有`key_part2`列。这个列也包含在索引`idx_key_part`中。

​		可以直接通过遍历`idx_key_part`索引的叶子节点的记录来比较`key_part2 = 'abc'`这个条件是否成立，把匹配成功的二级索引记录的`key_part1`, `key_part2`, `key_part3`列的值直接加到结果集中就行了。由于**二级索引记录比聚簇索记录小的多**（聚簇索引记录要存储所有用户定义的列以及所谓的隐藏列，而二级索引记录只需要存放索引列和主键），而且这个过程也**不用进行回表操作**，所以直接遍历二级索引比直接遍历聚簇索引的成本要小很多，设计`MySQL`的大佬就把这种采用遍历二级索引记录的执行方式称之为：`index`。

### all

最直接的查询执行方式就是全表扫描，对于`InnoDB`表来说也就是直接扫描聚簇索引。



注：

一个使用到索引的搜索条件和没有使用该索引的搜索条件使用`OR`连接起来后是无法使用该索引的。



### index merge

在一般情况下执行一个查询时最多只会用到单个二级索引。但在特殊情况下也可能在一个查询中使用到多个二级索引

#### Intersection合并

翻译为交集，对应不同索引搜索条件用`AND`连接的情况，某个查询可以使用多个二级索引，将从多个二级索引中查询到的结果取交集，比方说下面这个查询：

```sql
SELECT * 
FROM single_table 
WHERE key1 = 'a' AND key3 = 'b'
```

情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只匹配部分列的情况。

- 需要保证最终取出的主键值是排序的（此时时间复杂度为O(n)），否则取主键交集的时候还需要排序，时间复杂度会增加。

情况二：主键列可以是范围匹配

```sql
SELECT * 
FROM single_table 
WHERE id > 100 AND key1 = 'a';
```

> 小贴士：按照有序的主键值去回表取记录有个专有名词儿，叫：Rowid Ordered Retrieval，简称ROR，以后大家在某些地方见到这个名词儿就眼熟了。

​		上面说的`情况一`和`情况二`只是发生`Intersection`索引合并的必要条件，不是充分条件。也就是说即使情况一、情况二成立，也不一定发生`Intersection`索引合并，这得看优化器的心情。**优化器只有在单独根据搜索条件从某个二级索引中获取的记录数太多，导致回表开销太大，而通过`Intersection`索引合并后需要回表的记录数大大减少时才会使用`Intersection`索引合并**。

> 注：部分Intersection索引合并（比如情况一上面的语句）可以使用联合索引替代，还能只维护一个索引，简直又快又好

#### Union合并

翻译为并集，对应不同索引搜索条件用`OR`连接的情况

- 情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只出现匹配部分列的情况。

- 情况二：主键列可以是范围匹配

- 情况三：使用`Intersection`索引合并的搜索条件

    这种情况其实也挺好理解，就是搜索条件的某些部分使用`Intersection`索引合并的方式得到的主键集合和其他方式得到的主键集合取交集，比方说这个查询：

  ```sql
  SELECT * 
  FROM single_table 
  WHERE key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c' OR (key1 = 'a' AND key3 = 'b');
  ```

    优化器可能采用这样的方式来执行这个查询：

  - 先按照搜索条件`key1 = 'a' AND key3 = 'b'`从索引`idx_key1`和`idx_key3`中使用`Intersection`索引合并的方式得到一个主键集合。
  - 再按照搜索条件`key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c'`从联合索引`idx_key_part`中得到另一个主键集合。
  - 采用`Union`索引合并的方式把上述两个主键集合取并集，然后进行回表操作，将结果返回给用户。

​		当然，查询条件符合了这些情况也不一定就会采用`Union`索引合并，也得看优化器的心情。优化器只有在单独根据搜索条件从某个二级索引中获取的记录数比较少，通过`Union`索引合并后进行访问的代价比全表扫描更小时才会使用`Union`索引合并。

#### Sort-Union合并

​		先按照二级索引记录的主键值进行排序，之后按照`Union`索引合并方式执行的方式称之为`Sort-Union`索引合并，很显然，这种`Sort-Union`索引合并比单纯的`Union`索引合并多了一步对二级索引记录的主键值排序的过程。

```sql
SELECT * FROM single_table WHERE key1 < 'a' OR key3 > 'z'
```

根据`key1 < 'a'`从`idx_key1`索引中获取的二级索引记录的主键值不是排好序的，根据`key3 > 'z'`从`idx_key3`索引中获取的二级索引记录的主键值也不是排好序的，但是`key1 < 'a'`和`key3 > 'z'`这两个条件又特别让我们动心，所以我们可以这样：

- 先根据`key1 < 'a'`条件从`idx_key1`二级索引总获取记录，并按照记录的主键值进行排序
- 再根据`key3 > 'z'`条件从`idx_key3`二级索引总获取记录，并按照记录的主键值进行排序
- 因为上述的两个二级索引主键值都是排好序的，剩下的操作和`Union`索引合并方式就一样了。

> 小贴士：为什么有Sort-Union索引合并，就没有Sort-Intersection索引合并么？是的，的确没有Sort-Intersection索引合并这么一说，Sort-Union的适用场景是单独根据搜索条件从某个二级索引中获取的记录数比较少，这样即使对这些二级索引记录按照主键值进行排序的成本也不会太高，而Intersection索引合并的适用场景是单独根据搜索条件从某个二级索引中获取的记录数太多，导致回表开销太大，合并后可以明显降低回表开销，但是如果加入Sort-Intersection后，就需要为大量的二级索引记录按照主键值进行排序，这个成本可能比回表查询都高了，所以也就没有引入Sort-Intersection这个玩意儿。  

## 其他

### system

  当表中只有一条记录并且该表使用的存储引擎的统计数据是精确的，比如MyISAM、Memory，那么对该表的访问方法就是`system`。比方说我们新建一个`MyISAM`表，并为其插入一条记录：

```sql
mysql> CREATE TABLE t(i int) Engine=MyISAM;
Query OK, 0 rows affected (0.05 sec)

mysql> INSERT INTO t VALUES(1);
Query OK, 1 row affected (0.01 sec)
```

  然后我们看一下查询这个表的执行计划：

```sql
mysql> EXPLAIN SELECT * FROM t;
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

  可以看到`type`列的值就是`system`了。

​		如果是InnoDB存储引擎，那么`type`为ALL~

```sql
mysql> explain select * from (select * from t1 where m1 = 2) as t3
    -> ;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

### eq_ref

  在连接查询时，如果被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（如果该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较），则对该被驱动表的访问方法就是`eq_ref`，比方说：

```sql
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref             | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL            | 9688 |   100.00 | NULL  |
|  1 | SIMPLE      | s2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | xiaohaizi.s1.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+-----------------+------+----------+-------+
2 rows in set, 1 warning (0.01 sec)
```

  从执行计划的结果中可以看出，`MySQL`打算将`s1`作为驱动表，`s2`作为被驱动表，重点关注`s2`的访问方法是`eq_ref`，表明在访问`s2`表的时候可以通过主键的等值匹配来进行访问。

### fulltext

  全文索引，我们没有细讲过，跳过～

### unique_subquery

  类似于两表连接中被驱动表的`eq_ref`访问方法，`unique_subquery`是针对在一些包含`IN`子查询的查询语句中，如果查询优化器决定将`IN`子查询转换为`EXISTS`子查询，而且子查询可以使用到主键进行等值匹配的话，那么该子查询执行计划的`type`列的值就是`unique_subquery`，比如下面的这个查询语句：

```sql
mysql> EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type            | possible_keys    | key     | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL             | idx_key3         | NULL    | NULL    | NULL | 9688 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | unique_subquery | PRIMARY,idx_key1 | PRIMARY | 4       | func |    1 |    10.00 | Using where |
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)
```

  可以看到执行计划的第二条记录的`type`值就是`unique_subquery`，说明在执行子查询时会使用到`id`列的索引。

### index_subquery

  `index_subquery`与`unique_subquery`类似，只不过访问子查询中的表时使用的是普通的索引，比如这样：

```sql
mysql> EXPLAIN SELECT * FROM s1 WHERE common_field IN (SELECT key3 FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
+----+--------------------+-------+------------+----------------+-------------------+----------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type           | possible_keys     | key      | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+----------------+-------------------+----------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL            | idx_key3          | NULL     | NULL    | NULL | 9688 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | index_subquery | idx_key1,idx_key3 | idx_key3 | 303     | func |    1 |    10.00 | Using where |
+----+--------------------+-------+------------+----------------+-------------------+----------+---------+------+------+----------+-------------+
2 rows in set, 2 warnings (0.01 sec)
```











完整文章为：《MySQL 是怎样运行的：从根儿上理解 MySQL》——第10章 条条大路通罗马-单表访问方法

《MySQL 是怎样运行的：从根儿上理解 MySQL》——第15章 查询优化的百科全书-Explain详解（上）

以上只是一些自己想要记录且方便查找的笔记
