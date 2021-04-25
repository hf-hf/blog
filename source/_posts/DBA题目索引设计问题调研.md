---
title: DBA题目索引设计问题调研
tags:
  - Mysql
categories: uncategorized
cover: /upload/homePage/20210425115900.jpg
abbrlink: a4fbff2c
date: 2021-04-23 17:08:46
---
## 题目
本题目来源自网络，引用自[cnblogs 05 | 深入浅出索引（下）](https://www.cnblogs.com/gaosf/p/11142108.html)，最初出处因搜索结果过多目前无法确定。

DBA小吕在入职新公司的时候，就发现自己接手维护的库里面，有这么一个表，表结构定义类似这样的：

```
CREATE TABLE `geek` (
`a` int(11) NOT NULL,
`b` int(11) NOT NULL,
`c` int(11) NOT NULL,
`d` int(11) NOT NULL,
PRIMARY KEY (`a`,`b`),
KEY `c` (`c`),
KEY `ca` (`c`,`a`),
KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;
```

公司的同事告诉他说，由于历史原因，这个表需要a、b做联合主键，这个小吕理解了。

但是，学过本章内容的小吕又纳闷了，既然主键包含了a、b这两个字段，那意味着单独在字段c上创建一个索引，就已经包含了三个字段了呀，为什么要创建“ca”“cb”这两个索引？

同事告诉他，是因为他们的业务里面有这样的两种语句：

```
select * from geek where c=N order by a limit 1;
select * from geek where c=N order by b limit 1;
```
我给你的问题是，这位同事的解释对吗，为了这两个查询模式，这两个索引是否都是必须的？为什么呢？

## 问题思考
首先我们从select的条件来分析一下，这两条查询条件都是一样的唯一的区别是一个是根据a字段排序，另一个根据b字段排序。
从查询条件分析，因为都是c=N，该条件索引c、ca、cb都能匹配到（Mysql最左匹配原则）。

> 最左匹配原则：
> 最左前缀匹配原则, mysql会一直向右匹配直到遇到范围查询(>, <, between, like)就停止匹配, 比如a=1 and b=2 and c>3 and d=4 如果建立了(a,b,c,d)顺序> 的索引, d是用不到索引的, 如果建立(a,b,d,c)的索引, 则都可以使用到, a,b,d的顺序可以任意调整。
> = 和 in 可以乱序, 比如 a=1 and b=2 and c=3 建立(a,b,c)索引可以任意顺序, mysql 的查询优化器会帮你优化成索引可以识别的形式。

从排序条件分析，在a、b联合主键的前提下，该表的默认排序为order a,b，也就是先按a排序，在按b排序，其他字段无序。此时天然就满足第一个select的order a排序条件，因此ca索引是不必要的，见下方数据排序表格。

### 联合主键a，b下的数据排序
|a|b|c|d|
|-|-|-|-|
|1|2|3|d|
|1|3|2|d|
|1|4|3|d|
|2|1|3|d|
|2|2|2|d|
|2|3|4|d|

先按a排序，在按b排序，其他字段无序。

### 索引ca下的聚簇索引关联数据的排序
|c|a|b(主键部分)|d|
|-|-|-|-|
|2|1|3|d|
|2|2|2|d|
|3|1|2|d|
|3|1|4|d|
|3|2|1|d|
|4|2|3|d|

索引ca为非聚簇索引，其叶子节点为聚簇索引的索引值，该聚簇索引值已按ca来排序，通过该索引值可以在聚簇索引的叶子节点获取表数据，该表数据位于聚簇索引叶子节点，因此表数据默认就存在主键a,b排序，所以使用ca索引查询到的数据其排序顺序为：先按c排序，再按a排序，再记录主键b的排序（注意此处起作用的主键部分为b，而不是ab）。

此时索引ca和索引c的数据是一模一样的。

### 索引cb下的聚簇索引关联数据的排序
|c|b|a(主键部分)|d|
|-|-|-|-|
|2|2|2|d|
|2|3|1|d|
|3|1|2|d|
|3|2|1|d|
|3|4|1|d|
|4|3|2|d|

原因同上，使用cb索引查询到的数据其排序顺序为：先按c排序，在按b排序，再记录主键a的排序。

所以，结论是ca可以去掉，cb需要保留。

## InnoDB引擎索引说明
上面提到了聚簇索引和非聚簇索引，之前在[Mysql优化减少回表操作](https://hunfan.top/posts/4bdcd099/)一文中其实已经介绍过了，这里结合图文再拓展延伸一下。

### 聚簇索引
Mysql InnoDb每个表都有一个聚簇索引，其取值规则见下方：
- 主键存在时以主键为聚簇索引，
- 主键不存在时，以第一个不含有null值的唯一索引作为聚簇索引
- 以上索引都不存在时，MySQL会创建一个隐藏字段rowid的聚簇索引。
每个表的数据按照聚簇索引而聚集在一起形成B+树，其中在最后的叶子节点挂载非索引数据，叶子节点之间存在有序的指针。

![聚簇索引.png](/upload/DBA题目索引设计问题调研/聚簇索引.png) 

### 辅助索引
表中除了聚簇索引外其他非聚簇索引称为二级索引或者辅助索引，辅助索引中的叶子节点不再挂载非索引数据，而是存储聚簇索引的索引值。

![辅助索引.png](/upload/DBA题目索引设计问题调研/辅助索引.png) 

### 联合索引
特殊的辅助索引：联合索引，B+树的节点存储的不是一个列数据，而是多个列数据，按照定义的顺序构成一个节点。

![联合索引.png](/upload/DBA题目索引设计问题调研/联合索引.png) 

辅助索引和联合索引，可以统称为非聚簇索引，从图上可以看出，非聚簇索引的叶子节点存储是聚簇索引的索引值，但是其是按照非聚簇索引来排序的，因此我们使用合理的索引可以减少Mysql的Using filesort，原因就在这里。而数据存储在聚簇索引的叶子节点，所以我们通过非聚簇索引查询到聚簇索引值时，还需要再去聚簇索引上查找表数据，这就是所谓的回表操作，执行回表操作的数据量越少，我们相应查询消耗的时间也会越少。聚簇索引叶子节点的表数据，是根据聚簇索引来排序的，按照大部分的情况，聚簇索引都是表的主键，所以表数据默认是按照主键升序的，结合主键来创建索引，可以减少很多不必要的索引创建。但是我们要记住索引的创建还是根据真实的检索条件来的，比如一个有a,b,c三个字段的数据表，如果我们的最常使用的查询就是select xxx where a=n and b=n and c=n，那么我们只需要创建一个a,b,c的联合索引，尽量建立聚合索引而不是多个单索引，where条件后面按照聚合索引列作为条件，减少查询聚簇索引时匹配的数据行。

> ICP(index_condition_pushdown)
> 在索聚簇索引树查询数据行之前，匹配的数据行越少，越精确则查询效率越高。ICP(index_condition_pushdown)技术就是优化的这部分，旨在尽量减少数据行加载到> 内存中。在InnoDB引擎中ICP只支持联合索引，因为聚簇索引能直接锁定要查询的数据行，无法继续再筛选(聚簇索引只有一个索引)，而联合索引则是至少2个索引，在> 第一个索引匹配的行数和后续其他联合索引匹配的行数处理后，再回表到聚簇索引树中查询数据，这样聚簇索引树中的数据行就会缩减，从而提高效率。ICP技术是默认> 开启的。explain提示信息为：Using index condition，设置参数为：index_condition_pushdown。

## 实践出真知
运行环境为Mysql5.7.24。
```
# 查询Mysql版本
select version();
5.7.24
# 创建geek数据表
CREATE TABLE `geek` (
`a` int(11) NOT NULL,
`b` int(11) NOT NULL,
`c` int(11) NOT NULL,
`d` int(11) NOT NULL,
PRIMARY KEY (`a`,`b`),
KEY `c` (`c`),
KEY `ca` (`c`,`a`),
KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;
```

使用explain分析查询语句。
```
explain select * from geek where c=1 order by a limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek    ref    c,ca,cb       c       4     const    1     100.00  Using index condition

explain select * from geek where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek    ref    c,ca,cb       c       4     const    1     100.00  Using index condition; Using filesort

# 此时我们删掉ca索引
drop index ca on geek;
# 重新运行explain select...
explain select * from geek where c=1 order by a limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek   ref     c,cb          c       4     const    1     100.00  Using index condition

explain select * from geek where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek   ref     c,cb          c       4     const    1     100.00  Using index condition; Using filesort
# 可以看到最终的执行效果是没有变化的，ca索引是可以去掉的
# 但是我们看到order by b的查询语句，实际使用的是索引c，并且Extra中存在Using filesort，这说明其进行了额外的文件排序
# 我们强制让其使用cb索引看一下分析结果
explain select * from geek force index(cb) where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek   ref     c,cb          cb     4      const    1     100.00  Using index condition
# 可以看到Using filesort没有了，但是为什么不强制使用索引cb，Mysql就不会选择到最优的cb索引来执行查询呢？
```

## 拓展思考
通过实际运行explain分析查询的select语句来验证了我们的问题思考，但是在最后又产生了一个新的问题，我们发现执行select xxx order by b语句时，若不强制其使用cb索引，实际只会使用索引c，导致explain的分析中显示其在查询到结果数据后又使用了文件排序（Using filesort）。

现在我们通过对比测试，来分析一下具体是什么原因导致的产生这个问题。
```
# 首先我们删除原来的测试表geek
drop table geek;
# 移除原建表语句中的主键b以及索引ca，创建数据表
CREATE TABLE `geek` (
`a` int(11) NOT NULL,
`b` int(11) NOT NULL,
`c` int(11) NOT NULL,
`d` int(11) NOT NULL,
PRIMARY KEY (`a`),
KEY `c` (`c`),
KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;
# 运行explain select...
explain select * from geek where c=1 order by a limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek   ref    c,cb           c      4      const    1     100.00  Using index condition

explain select * from geek where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1    SIMPLE      geek   ref    c,cb           cb     4      const    1     100.00  Using index condition
# 可以看到在只有主键a的情况下，分析执行两条select...，其都匹配到了最佳的索引
# order by b，这一条也显示使用了索引cb
# 目前我们和上一节中运行的唯一区别就是由原来的联合主键a,b改为了主键a，移除了主键b
# 为了排除是否为联合索引引起的问题，这边又测试了改为联合主键a,c、联合主键a,d的两种情况
# 这里省略的修改数据表的语句，其分析结果见下方
# 联合主键a,c
explain select * from geek where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered  Extra
1   SIMPLE       geek   ref    c,cb           cb     4      const    1     100.00  Using index condition
# 联合主键a,d
explain select * from geek where c=1 order by b limit 1;
id  select_type  table  type  possible_keys   key  key_len  ref    rows  filtered   Extra
1   SIMPLE       geek   ref     c,cb          cb     4      const    1     100.00   Using where; Using index
# 从现在的结果可以看出，除了联合主键a,b的情况下会出现Using filesort并只匹配到索引c，其他诸如主键a、主键a,c、主键a,d均能匹配到索引cb
```

由此我们可以得出结论，只有b字段参与联合主键的情况下，才会导致select * from geek where c=1 order by b limit 1，未能匹配到索引cb，仅能匹配到索引c。那么根据之前了解到的Mysql InnoDb索引原理以及B+树的结构，我们可以推测select * from geek where c=1 order by b limit 1，其根据非聚簇索引（辅助索引）c查询到叶子节点（聚簇索引），使用聚簇索引值去聚簇索引树中查询表数据，发现聚簇索引a,b包含排序条件b，此时就直接在当前数据下进行文件排序并返回结果，因此才会产生Using filesort的出现。
我这边推测其产生的原因在于聚簇索引下的表数据，本身就是以联合主键a，b排序，此时要返回order by b的数据，直接在其基础上以排序算法来进行文件排序，效率可能比匹配cb要更优，或者这个干脆就是Mysql当前版本下的一个bug，如果有知道详细原因的朋友，不吝赐教。

## 参考资料
[05 | 深入浅出索引（下）](https://www.cnblogs.com/gaosf/p/11142108.html)
[06 | 全局锁和表锁 ：给表加个字段怎么有这么多阻碍？](https://www.cnblogs.com/gaosf/p/11142112.html)
[mysql 联合主键_搞懂MySQL中的SQL优化，就靠这篇文章了](https://blog.csdn.net/weixin_39918682/article/details/111278002)
[多个索引时，mysql索引的命中规则](https://blog.csdn.net/qq_26975307/article/details/91367232)