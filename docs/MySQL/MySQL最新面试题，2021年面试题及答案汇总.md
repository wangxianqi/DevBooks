# MySQL最新面试题，2021年面试题及答案汇总

### 其实，博主还整理了，更多大厂面试题，直接下载吧

### 下载链接：[高清172份，累计 7701 页大厂面试题  PDF](https://github.com/souyunku/DevBooks/blob/master/docs/index.md)



### 1、MySQL数据库cpu飙升的话，要怎么处理呢？

**排查过程：**

**1、** 使用top 命令观察，确定是MySQLd导致还是其他原因。

**2、** 如果是MySQLd导致的，show processlist，查看session情况，确定是不是有消耗资源的sql在运行。

**3、** 找出消耗高的 sql，看看执行计划是否准确， 索引是否缺失，数据量是否太大。

**处理：**

**1、** kill 掉这些线程(同时观察 cpu 使用率是否下降)，

**2、** 进行相应的调整(比如说加索引、改 sql、改内存参数)

**3、** 重新跑这些 SQL。

**其他情况：**

也有可能是每个 sql 消耗资源并不多，但是突然之间，有大量的 session 连进来导致 cpu 飙升，这种情况就需要跟应用一起来分析为何连接数会激增，再做出相应的调整，比如说限制连接数等


### 2、说说对SQL语句优化有哪些方法？（选择几条）

**1、** Where子句中：where表之间的连接必须写在其他Where条件之前，那些可以过滤掉最大数量记录的条件必须写在Where子句的末尾.HAVING最后。

**2、** 用EXISTS替代IN、用NOT EXISTS替代NOT IN。

**3、**  避免在索引列上使用计算

**4、** 避免在索引列上使用IS NULL和IS NOT NULL

**5、** 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

**6、** 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描

**7、** 应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描


### 3、Innodb的事务与日志的实现方式

**有多少种日志**

innodb两种日志redo和undo。

**日志的存放形式**

**1、** redo：在页修改的时候，先写到 redo log buffer 里面， 然后写到 redo log 的文件系统缓存里面(fwrite)，然后再同步到磁盘文件（ fsync）。

**2、** Undo：在 MySQL5.5 之前， undo 只能存放在 ibdata文件里面， 5.6 之后，可以通过设置 innodb_undo_tablespaces 参数把 undo log 存放在 ibdata之外。

**事务是如何通过日志来实现的**

**1、** 因为事务在修改页时，要先记 undo，在记 undo 之前要记 undo 的 redo， 然后修改数据页，再记数据页修改的 redo。 Redo（里面包括 undo 的修改） 一定要比数据页先持久化到磁盘。

**2、** 当事务需要回滚时，因为有 undo，可以把数据页回滚到前镜像的 状态，崩溃恢复时，如果 redo log 中事务没有对应的 commit 记录，那么需要用 undo把该事务的修改回滚到事务开始之前。

**3、** 如果有 commit 记录，就用 redo 前滚到该事务完成时并提交掉。


### 4、非聚簇索引一定会回表查询吗？

不一定，如果查询语句的字段全部命中了索引，那么就不必再进行回表查询（哈哈，覆盖索引就是这么回事）。

举个简单的例子，假设我们在学生表的上建立了索引，那么当进行select age from student where age < 20的查询时，在索引的叶子节点上，已经包含了age信息，不会再次进行回表查询。


### 5、Hash索引和B+树所有有什么区别或者说优劣呢?

**1、** 首先要知道Hash索引和B+树索引的底层实现原理：

**2、** hash索引底层就是hash表，进行查找时，调用一次hash函数就可以获取到相应的键值，之后进行回表查询获得实际数据。B+树底层实现是多路平衡查找树。对于每一次的查询都是从根节点出发，查找到叶子节点方可以获得所查键值，然后根据查询判断是否需要回表查询数据。

**那么可以看出他们有以下的不同：**

**1、** hash索引进行等值查询更快(一般情况下)，但是却无法进行范围查询。

**2、** 因为在hash索引中经过hash函数建立索引之后，索引的顺序与原顺序无法保持一致，不能支持范围查询。而B+树的的所有节点皆遵循(左节点小于父节点，右节点大于父节点，多叉树也类似)，天然支持范围。

**3、** hash索引不支持使用索引进行排序，原理同上。

**4、** hash索引不支持模糊查询以及多列索引的最左前缀匹配。原理也是因为hash函数的不可预测。AAAA和AAAAB的索引没有相关性。

**5、** hash索引任何时候都避免不了回表查询数据，而B+树在符合某些条件(聚簇索引，覆盖索引等)的时候可以只通过索引完成查询。

**6、** hash索引虽然在等值查询上较快，但是不稳定。性能不可预测，当某个键值存在大量重复的时候，发生hash碰撞，此时效率可能极差。而B+树的查询效率比较稳定，对于所有的查询都是从根节点到叶子节点，且树的高度较低。

**7、** 因此，在大多数情况下，直接选择B+树索引可以获得稳定且较好的查询速度。而不需要使用hash索引。


### 6、select for update有什么含义，会锁表还是锁行还是其他。

**select for update 含义**

select查询语句是不会加锁的，但是select for update除了有查询的作用外，还会加锁呢，而且它是悲观锁哦。至于加了是行锁还是表锁，这就要看是不是用了索引/主键啦。

没用索引/主键的话就是表锁，否则就是是行锁。

**select for update 加锁验证**

表结构：

```
//id 为主键，name为唯一索引
CREATE TABLE `account` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `balance` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1570068 DEFAULT CHARSET=utf8
```

id为主键，select for update 1270070这条记录时，再开一个事务对该记录更新，发现更新阻塞啦，其实是加锁了。如下图：

![](https://user-gold-cdn.xitu.io/2020/5/23/1723f1df3f7cd93d?w=1338&h=337&f=png&s=47172#alt=)

我们再开一个事务对另外一条记录1270071更新，发现更新成功，因此，如果查询条件用了索引/主键，**会加行锁**~

![](https://user-gold-cdn.xitu.io/2020/5/23/1723f21ad2567560?w=1419&h=315&f=png&s=54135#alt=)

我们继续一路向北吧，换普通字段balance吧，发现又阻塞了。因此，没用索引/主键的话，**select for update加的就是表锁**

![](https://user-gold-cdn.xitu.io/2020/5/23/1723f297aeeb95d5?w=1421&h=384&f=png&s=56635#alt=)


### 7、你们数据库是否支持emoji表情存储，如果不支持，如何操作？

更换字符集utf8-->utf8mb4


### 8、索引的数据结构（b树，hash）

索引的数据结构和具体存储引擎的实现有关，在MySQL中使用较多的索引有**Hash索引**，**B+树索引**等，而我们经常使用的InnoDB存储引擎的默认索引实现为：B+树索引。对于哈希索引来说，底层的数据结构就是哈希表，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；其余大部分场景，建议选择BTree索引。

**B树索引**

MySQL通过存储引擎取数据，基本上90%的人用的就是InnoDB了，按照实现方式分，InnoDB的索引类型目前只有两种：BTREE（B树）索引和HASH索引。B树索引是MySQL数据库中使用最频繁的索引类型，基本所有存储引擎都支持BTree索引。通常我们说的索引不出意外指的就是（B树）索引（实际是用B+树实现的，因为在查看表索引时，MySQL一律打印BTREE，所以简称为B树索引）

![99_1.png][99_1.png]

**查询方式：**

**1、** 主键索引区:PI(关联保存的时数据的地址)按主键查询,

**2、** 普通索引区:si(关联的id的地址,然后再到达上面的地址)。所以按主键查询,速度最快

**B+tree性质：**

**1、** n棵子tree的节点包含n个关键字，不用来保存数据而是保存数据的索引。

**2、** 所有的叶子结点中包含了全部关键字的信息，及指向含这些关键字记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。

**3、** 所有的非终端结点可以看成是索引部分，结点中仅含其子树中的最大（或最小）关键字。

**4、** B+ 树中，数据对象的插入和删除仅在叶节点上进行。

**5、** B+树有2个头指针，一个是树的根节点，一个是最小关键码的叶节点。

**哈希索引**

简要说下，类似于数据结构中简单实现的HASH表（散列表）一样，当我们在MySQL中用哈希索引时，主要就是通过Hash算法（常见的Hash算法有直接定址法、平方取中法、折叠法、除数取余法、随机数法），将数据库字段数据转换成定长的Hash值，与这条数据的行指针一并存入Hash表的对应位置；如果发生Hash碰撞（两个不同关键字的Hash值相同），则在对应Hash键下以链表形式存储。当然这只是简略模拟图。

![99_2.png][99_2.png]


### 9、最左匹配原则？

在创建联合索引时候，一般需要遵循最左匹配原则。即联合索引中的属性识别度最高的放在查询语句的最前面。


### 10、对于关系型数据库而言，索引是相当重要的概念，请回答有关索引的几个问题：

1.索引的目的是什么？

快速访问数据表中的特定信息，提高检索速度

创建唯一性索引，保证数据库表中每一行数据的唯一性。

加速表和表之间的连接

使用分组和排序子句进行数据检索时，可以显著减少查询中分组和排序的时间

2.索引对数据库系统的负面影响是什么？

负面影响：

创建索引和维护索引需要耗费时间，这个时间随着数据量的增加而增加；索引需要占用物理空间，不光是表需要占用数据空间，每个索引也需要占用物理空间；当对表进行增、删、改、的时候索引也要动态维护，这样就降低了数据的维护速度。

3.为数据表建立索引的原则有哪些？

在最频繁使用的、用以缩小查询范围的字段上建立索引。

在频繁使用的、需要排序的字段上建立索引

4.什么情况下不宜建立索引？

对于查询中很少涉及的列或者重复值比较多的列，不宜建立索引。

对于一些特殊的数据类型，不宜建立索引，比如文本字段（text）等


### 11、关心过业务系统里面的sql耗时吗？统计过慢查询吗？对慢查询都怎么优化过？
### 12、如何优化长难的查询语句
### 13、联合索引是什么？为什么需要注意联合索引中的顺序？
### 14、什么是通用SQL函数？
### 15、谈谈MySQL的Explain
### 16、MySQL 遇到过死锁问题吗，你是如何解决的？
### 17、优化LIMIT分页
### 18、UNION与UNION ALL的区别？
### 19、什么是游标？
### 20、什么是子查询
### 21、MySQL 分页
### 22、limit 1000000 加载很慢的话，你是怎么解决的呢？
### 23、Innodb的事务与日志的实现方式
### 24、索引是什么?
### 25、SQL的生命周期？
### 26、锁的优化策略
### 27、读写分离常见方案？
### 28、什么是SQL？
### 29、MySQL存储引擎MyISAM与InnoDB区别
### 30、MYSQL的主从延迟，你怎么解决？




## 全部答案，整理好了，直接下载吧

### 下载链接：[全部答案，整理好了](https://www.souyunku.com/wp-content/uploads/weixin/githup-weixin-2.png)




## 最新，高清PDF：172份，7701页，最新整理

[![大厂面试题](https://www.souyunku.com/wp-content/uploads/weixin/mst.png "架构师专栏")](https://www.souyunku.com/wp-content/uploads/weixin/githup-weixin.png "架构师专栏")

[![大厂面试题](https://www.souyunku.com/wp-content/uploads/weixin/githup-weixin.png "架构师专栏")](https://www.souyunku.com/wp-content/uploads/weixin/githup-weixin.png "架构师专栏")
