---
title: 浅谈 MySQL 的连表查询

date: 2020-03-27T09:01:26+08:00

tag: [ MySQL, 数据库, explain, 连表查询]

categories: MySQL
description: 关于mysql的连表查询
---

#  浅谈 MySQL 的 连表查询

昨天看了一个下午的`MySQL` ，总算学到点知识，晚上陪老爸看中国机长看的太晚，就早早的睡了，早上起床乘着思路还清晰，赶紧记录一点东西。



## 是什么 `EXPLAIN` 语句 

>  The [`EXPLAIN`](https://dev.mysql.com/doc/refman/8.0/en/explain.html) statement provides information about how MySQL executes statements. [`EXPLAIN`](https://dev.mysql.com/doc/refman/8.0/en/explain.html) works with [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html), [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html), [`INSERT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html), [`REPLACE`](https://dev.mysql.com/doc/refman/8.0/en/replace.html), and [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) statements.
>
> [`EXPLAIN`](https://dev.mysql.com/doc/refman/8.0/en/explain.html) returns a row of information for each table used in the [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statement. It lists the tables in the output in the order that MySQL would read them while processing the statement. MySQL resolves all joins using a nested-loop join method. This means that MySQL reads a row from the first table, and then finds a matching row in the second table, the third table, and so on. When all tables are processed, MySQL outputs the selected columns and backtracks through the table list until a table is found for which there are more matching rows. The next row is read from this table and the process continues with the next table.

这段话引用自 [MySQL 官方手册][1]，

[1]: https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

意思是  `EXPLAIN` 语句提供了有关 MySQL 执行状态的有关信息.  `EXPLAIN` 语句能够解释  `SELECT` , `DELETE` ,` INSERT` , `REPLACE` 和  `UPDATE`  语句.   

`EXPLAIN` 会对  `SELECT` 使用的每一个表返回一行信息, 它会按照 MySQL 在处理 他们的时候按照处理的顺序列出输出的表.  MySQL 使用循环嵌套的方法来解析所有的 `JOIN` . 意思是 MySQL 从第一个表读取一行,  然后从第二个表匹配到符合要求的列, 然后再第三个表, 以此类推, 当所有的表都处理完时 MySQL 会  输出你选择的列且回溯这些表, 直到找到一个表存在更多的匹配的行. 然后再从表中读取下一行, 再处理下一个表

### MySQL 是如何处理连表的

可能上面的官方指南说起来可能太抽象了, 我就稍微自己说说:

**文字表述**

1.  从第一个表中找到所需要的行,假设有两行

2.  取第一个表的第一行, 来匹配第二个表 , 匹配到三行

3.  第一个表第一行匹配完毕，返回选择的列

4.  取第一个表的第二行, 来匹配第二个表, 匹配到三行

5. 第一个表的第二行匹配完毕，返回选择的列

6. 第一个表匹配完毕 , 返回结果

7. 关联更多表的处理逻辑也是一样的

   

**伪代码**

```java
List tmp_tb1 = tb1 中 匹配到的两行;
for (tmp_tb1_row : tmp_tb1) {
    List tmp_tb2 = tb2 中根据tmp_tb1_row 中 匹配到的行;
    for (tmp_tb2_row: tmp_tb2) {
        System.out.println(tmp_tb1_row.col1, mp_tb2_row.col2);
    }       
}
```



**图片说明**

![](https://guoxy-common.oss-cn-hangzhou.aliyuncs.com/mysql1.png)

![](https://guoxy-common.oss-cn-hangzhou.aliyuncs.com/sql2.png)

## EXPLAIN 的参数

| id     | select_type  | table | type     | possible_keys  | key                | key_len  | ref        | rows               | Exrta    |
| ------ | :----------: | ----- | -------- | -------------- | ------------------ | -------- | ---------- | ------------------ | -------- |
| 标识符 | select的类型 | 表名  | 连接类型 | 可能用到的索引 | 实际决定使用的索引 | 索引长度 | 关联用的列 | 查询需要扫描的行数 | 其他信息 |

 

+ 对于`InnoDB` rows 为估计的行数 



其他的参数很好理解，现在主要 `type` 和 `Extra` 两个参数

**type**

|      |       |       |      |        |               |      |
| ---- | ----- | ----- | ---- | ------ | ------------- | ---- |
| ALL  | Index | range | ref  | eq_ref | const, system | NULL |

 *从左至右：性能由最差到最好*

+ `ALL` ：Full Table Scan， MySQL将遍历全表以找到匹配的行

+ `index` ：Full Index Scan，index与ALL区别为index类型只遍历索引树

+ `range` ：索引范围扫描，对索引的扫描开始于某一点，返回匹配值域的行，常见于between、<、>等的查询

+ `ref` ：非唯一性索引扫描，返回匹配某个单独值的所有行。常见于使用非唯一索引即唯一索引的非唯一前缀进行的查找 

+ `eq_ref` ：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描 

+ `const` 、`system` ：当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量

+ `NULL` ：MySQL在优化过程中分解语句，执行时甚至不用访问表或索引

**Extra**

+ `Using temporary` :  表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询
+ `Using filesort` :  MySQL中无法利用索引完成的排序操作称为“文件排序”
+ `Using Index` : 表示直接访问索引就能够获取到所需要的数据（覆盖索引），不需要通过索引回表
+ `Using Index Condition` : 在MySQL 5.6版本后加入的新特性（Index Condition Pushdown）;会先条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行
+ `Using where` : 表示MySQL服务器在存储引擎收到记录后进行“后过滤”（Post-filter）,如果查询未能使用索引，Using where的作用只是提醒我们MySQL将用where子句来过滤结果集。这个一般发生在MySQL服务器，而不是存储引擎层。*一般发生在不能走索引扫描的情况下或者走索引扫描*，*但是有些查询条件不在索引当中的情况下*

## 举个栗子

这里有一段查询SQL:

```mysql
explain select xm,  bjmc, zymc, yxsmc
		from xs_jbxx
            left join xs_xqbj xx on xs_jbxx.xh = xx.xh
            left join xj_xqbj x on xx.xqbjid = x.id
            left join bj_jbxx bj on x.bh = bj.bh
            left join zy_jbxx zj on bj.zyh = zj.zyh
            left join yxs_jbxx yj on zj.yxsh = yj.yxsh
    where xqdm = '2019-2020(2)' and bjmc = '计算机类181班' ;
```

这是一个查询 计算机类181班 2019-2020(2) 学期的 所有学生姓名, 班级名称, 专业名称, 学院名称 的 explain

下面是结果:

| id   | select_type | table   | type   | possible_keys   | key     | key_len | ref     | rows | Extra                 |
| ---- | ----------- | ------- | ------ | --------------- | ------- | ------- | ------- | ---- | --------------------- |
| 1    | SIMPLE      | x       | ref    | PRIMARY,bh,xqdm | xqdm    | 39      | const   | 1124 | Using index condition |
| 1    | SIMPLE      | bj      | eq_ref | PRIMARY         | PRIMARY | 32      | x.bh    | 1    |                       |
| 1    | SIMPLE      | zj      | eq_ref | PRIMARY         | PRIMARY | 26      | bj.zyh  | 1    | Using where           |
| 1    | SIMPLE      | yj      | eq_ref | PRIMARY         | PRIMARY | 20      | zj.yxsh | 20   | Using where           |
| 1    | SIMPLE      | xx      | ref    | xqbjid,xh       | xqbjid  | 5       | x.id    | 1    |                       |
| 1    | SIMPLE      | xs_jbxx | eq_ref | PRIMARY         | PRIMARY | 47      | xx.xh   | 1    |                       |



+ 按照结果行的排序方式就是 MyAQL 的 连表顺序 , x 表就是主表(第一个表), 然后按照上面所说的 先连bj, 再连zj, 以此类推~
+ 第一行rows 1124 表示的大概扫描多少行, 刚好对应了 这个学期 有多少个班级,  type 为 ref 是因为 靠条件并不能获得唯一的匹配记录
+ 第五行的type 为ref 也是一样的原因,  一个学号对应了多个学期班级，也不能获得唯一值
+ MySQL 的查询顺序并不是按照你写的 SQL 语句的顺序， MySQL 内部有查询优化器，它会判断哪个作为主表是最优解

参考:

> [MySQL 官方手册](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
>
> *高性能MySQL*