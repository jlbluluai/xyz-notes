# Table of Contents

* [MySQL索引原理解读](#mysql索引原理解读)
    * [索引是啥？](#索引是啥？)
    * [索引的优势是啥？](#索引的优势是啥？)
    * [InnoDB索引模型](#innodb索引模型)
    * [索引设计](#索引设计)
        * [覆盖索引](#覆盖索引)
        * [最左前缀原则](#最左前缀原则)
        * [索引下推](#索引下推)


# MySQL索引原理解读

## 索引是啥？

> 在关系数据库中，索引是一种单独的、物理的对数据库表中一列或多列的值进行排序的一种存储结构，它是某个表中一列或若干列值的集合和相应的指向表中物理标识这些值的数据页的逻辑指针清单。索引的作用相当于图书的目录，可以根据目录中的页码快速找到所需的内容。

简单来说，字典有不同的目录（拼音、偏旁），都能查到字，索引就是数据库将你关注的列独立出来成为目录，我现在知道了这个列的data，我就可以用这个目录快速的查到这个数据。


## 索引的优势是啥？

简单明了，快！

例如你在新华字典里查`我`字，没有目录时怎么办？从第一个字开始找。其实我们知道新华字典的字本身是按首拼音排序的，那`w`可是排在很后的，你这样找估计要找到天荒地老。而根据目录找，假设我们这里按拼音目录，先定位到`w`，这就对比从第一页翻缩小了极大的范围了。

索引就能极大的加快这个查找速度，极端点有张订单表有1000w数据，用户张三有一条订单在里面不知道哪个咯吱窝，我们要找到张三的订单，没有索引怎么找，从第一条开始，比对用户id是不是张三，不是继续下一条，是捞出来，结束了吗？没有，我们要知道张三的所有订单，但我们不知道张三有几条订单，所以就算第一条就是张三的，也会继续找下去，最后就是找了1000w条数据。是不是有种海底捞针的感觉？

但现在不一样了，我们给用户id加了索引。会有什么不一样的效果？我们找到张三下有一条id为xxxxx的订单，ok，直接拿订单表的主键xxxxx回表查询，请问找了几条数据？非常肯定，1条。

这么极端的案例去理解，就知道索引的优势在哪。

## InnoDB索引模型

在InnoDB中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。所以本质上，每张表本身就自带一个索引——主键索引。

那么，索引的存在形式是啥？如果单纯的线性表，也难以想象会有速度的飞跃提升。所以当然不可能是线性表，也不是普通的树，而是B+树。

我们假设有张表中R1~R5的(ID,k)对应(100,1)(200,2)(300,3)(500,5)(600,6)，ID是主键，k也建有索引。针对这两棵索引树的示意图如下。

![](http://img.yelizi.top/06939f1c-517a-4cdc-9c05-313a96e40e6a.jpg$xyz)


以如下SQL为例：

```
select * from T where k=5;
```

由于k有索引，该查询会先走k索引树，查到k=5的id有500，这时会带着id=500回主表继续搜索，就能直接命中了。

InnoDB中，主键索引又称为聚簇索引，普通索引又称为二级索引。

[B+树浅解读参考](https://zhuanlan.zhihu.com/p/54102723)

## 索引设计

### 覆盖索引

复用上面举例的表：

```
select * from T where k=3;
```

这条SQL的执行流程是啥样的呢？

1. 在k索引树z找到k=3的记录，取得ID=300；
2. 再到ID索引树找到ID=300的记录R3；
3. 在k索引树娶下一个值k=5，不满足条件，结束（涉及到B+树按序排序的问题）。

这边其实是做了回表查询的操作，虽然通过索引快速定位快了不少，可能不能在什么情况下更快呢？

答案是肯定的，这就是覆盖索引的概念，改变上面的查询方式：

```
select ID,k from T where k=3;
```

这条查询不用回表，为啥呢？因为索引本身储藏的信息（主键，索引的字段的值）已经能够覆盖我们查询需要的结果了。

所以查询不是说无脑来个`*`就完事的，如果能合理利用覆盖索引，有些常见简直是飞速的提升性能。

**小提示**： 为了某种场景的查询快速，可以适当的建立联合索引，荣誉1~2个索引字段，这样就可以通过覆盖索引没有了回表。当然，该表必须是写次数少的，不然索引的维护成本将会很高，就得不偿失了。（当然类似这样的读多写少的常见，搞个缓存会更佳）

### 最左前缀原则

这个原则可以说是InnoDB最精妙的地方。常说维护索引成本高，但是有时候又得维护这么多索引，怎么在某种条件下能够减少些索引建立也能不放弃性能的提升呢？这就是最左前缀原则。

该原则规定了索引(A,B,C)在查询条件是A或A,B也生效，若是你有这些索引的需求就没必要再建立单独的。

所以我们在建立联合索引时，一定要考虑清楚了索引字段的顺序，恰当的顺序可以少维护索引都说不定。

**小提示**： 假设索引(A,B)，我们还需要单独的A索引和B索引，该如何选择(A,B)索引的顺序呢，好想随意就好，其实也可以优化。假设A的字段的大，就单独维护B索引，反之则反。为啥呢，字段大，索引维护的成本就大呗，稍微转念一想是不是就想明白了。


### 索引下推

我们先假设一家`user`表，里面有俩字段`name`和`age`，并且针对这俩字段创建一个联合索引:
```
`idx_name_age` (`name`,`age`)
```

我们进行如下查询：

```
select * from user where name like '李%' and age = 25;
```

由于like的前缀索引规则，所以这条DQL还是可以走到索引`idx_name_age`的。重点来了，可能大多数人会认为这里走的应该是联合索引的整个也就是`name`和`age`直接通过索引就能筛选出符合的再去回表。

这么理解也没问题，因为MySQL5.6开始看似貌似就是这样（而MySQL5.6开始才是目前市场主要的版本），其实这里发挥作用的就是索引下推，而这是5.6版本才引入了。

那么也就是5.6版本之前只能走单一的`name`索引，为什么呢？

> MySQL 会一直向右匹配直到遇到范围查询（>、<、between、like）就停止匹配。范围列可以用到索引，但是范围列后面的列无法用到索引。

那么我们这里一直向右匹配遇到了范围查询`like`，就停止了匹配，因此只会用到`name`的索引。


索引下推又是什么呢？

>【索引下推】Index Condition Pushdown，简称 ICP。 是Mysql 5.6版本引入的技术优化。旨在 在“仅能利用最左前缀索的场景”下（而不是能利用全部联合索引），对不在最左前缀索引中的其他联合索引字段加以利用——在遍历索引时，就用这些其他字段进行过滤(where条件里的匹配)。过滤会减少遍历索引查出的主键条数，从而减少回表次数，提示整体性能。 ------------------ 如果查询利用到了索引下推ICP技术，在Explain输出的Extra字段中会有“Using index condition”。即代表本次查询会利用到索引，且会利用到索引下推。 ------------------ 索引下推技术的实现——在遍历索引的那一步，由只传入可以利用到的字段值，改成了多传入下推字段值。

简单来说，索引由于规则，最终还是只用了`name`，但是联合索引中若有查询的字段，比如这里的`age`，就会拿来过滤查出的主键条数（回表次数），就是这么个优化。