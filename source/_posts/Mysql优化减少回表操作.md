---
title: Mysql优化减少回表操作
tags:
  - Mysql
cover: /upload/homePage/20210414152100.jpg
abbrlink: 4bdcd099
categories: uncategorized
date: 2021-04-14 10:19:12
---
## 情景
有一张日增量的Mysql InnoDB数据表，未分库分表，ID为自增主键，DAY_TAG为非唯一索引，当前数据量为69660129，现需根据其数据做每日的统计报表，每日数据分段统计分析后写入另一张表，数据分段查询使用limit offset,rows。
在此环境下，执行'select * from table where day_tag = '2021-04-14' limit 20, 2'只需要几毫秒结果就返回了，但是若执行'select * from table where day_tag = '2021-04-14' limit 300000, 2'，还是2条记录却会发现实际耗时很久。

## 问题分析
虽然我们仅需2条记录，但实际Mysql对前300000条记录都进行了不必要的回表操作，导致耗时增加，我们只要减少回表操作就可以有效的优化查询的效率。

那么什么是回表操作呢？在这之前我们要先了解两个概念聚簇索引和非聚簇索引。

- 聚集索引（聚簇索引）：以 InnoDB 作为存储引擎的表，表中的数据都会有一个主键，即使你不创建主键，系统也会帮你创建一个隐式的主键。这是因为 InnoDB 是把数据存放在 B+ 树中的，而 B+ 树的键值就是主键，在 B+ 树的叶子节点中，存储了表中所有的数据。这种以主键作为 B+ 树索引的键值而构建的 B+ 树索引，我们称之为聚集索引。
- 非聚集索引（非聚簇索引）：以主键以外的列值作为键值构建的 B+ 树索引，我们称之为非聚集索引。非聚集索引与聚集索引的区别在于非聚集索引的叶子节点不存储表中的数据，而是存储该列对应的主键，想要查找数据我们还需要根据主键再去聚集索引中进行查找，这个再根据聚集索引查找数据的过程，我们称为回表。

再代入我们的情景中，数据表的DAY_TAG就是非聚簇索引，所以如果我们执行'select id from table where day_tag = '2021-04-14' limit 300000, 2'，这时候是不会产生回表操作的，因为非聚簇索引的叶子节点存储的就是该列对应的主键ID；若执行'select * from table where day_tag = '2021-04-14' limit 300000, 2'，因为需要获取所有列的字段，所以查询到非聚簇索引的叶子节点数据后，需要再根据叶子节点上的主键值去聚簇索引上查询需要的全部字段值，见下图。

![Mysql优化减少回表操作_1](/upload/Mysql优化减少回表操作/Mysql优化减少回表操作_1.jpg)

像上图这样，'select * from table where day_tag = '2021-04-14' limit 300000, 2'实际Mysql需要查询300002次非聚簇索引，再查询300002次聚簇索引的数据，最后将结果过滤掉前300000条，取出最后2条。Mysql耗费了大量时间在查询聚簇索引的数据上，而且其中300000次查询到的数据还是不在结果集中出现的。

## 解决方案
我们可以采用子查询的方式，先查询出主键过滤后保留2条，再根据这2条的主键去聚簇索引上获取其他列的数据，修改后的sql见下方。
```
select * from table a 
inner join 
(select id from table
where day_tag = '2021-04-14'
limit 300000, 2) b
on a.id = b.id
```

优化后的执行过程见下图。

![Mysql优化减少回表操作_2](/upload/Mysql优化减少回表操作/Mysql优化减少回表操作_2.png)

## 参考资料
[mysql-order-by-limit-performance-late-row-lookups](https://explainextended.com/2009/10/23/mysql-order-by-limit-performance-late-row-lookups/)


