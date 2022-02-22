title: Postgresql select count性能与Vacuum
date: 2021-07-29 15:37:18
tags:
- postgresql
categories:
- DataBase
---

最近在开发过程中遇到一个postgresql的问题，某张表大多数时候只有insert及select操作，该表建立了一个非聚集索引，但是通过该索引进行select count操作时，时间长达数十秒。通过查找资料，我们已经定位到了问题所在，这次我们就谈谈Postgresql的元组与vacuum机制。


## Dead tuples(死元组)
postgresql的表的一行数据通常被称为元组(tuple)，由于表自带row version功能的实现，当你在postgresql中执行DELETE时，行数据不会立即从数据文件中删除，而是仅通过在页头中设置xmax字段将其标记为已删除。同样对于UPDATE,它可能在postgresql中被视为DELETE+INSERT。这也导致了表中的元组数据既包含了可见元组，也包含了不可见元组（通常称为死元组）。

如果没有清理，那些"死元组"(对于任何事务实际上是不可见的)将永远留在数据文件中。对于DELETE和UPDATE比较多的的表，死元组可能占据很多磁盘空间。同时，死元组也将从索引中引用，进一步增加了浪费的磁盘空间量，同时因为查询也会变慢。

当我们进行count查询时，如果查询数据范围足够小，就会直接走索引扫描，但如果查询的数据范围比较大，postgresql就会花费很长时间遍历元组堆数据，来筛选出可见元组和不可见死元组，这是postgresql平衡磁盘IO和内存压力的一种机制。

那么我们怎么优化呢？

### Autovacuum
postgresql存在自动清理元组的功能，也就是autovacuum后台任务。当表的更新与删除操作达到一定阈值时，便会触发自动清理死元组，或者分析最佳查询计划，autovacuum会受到以下参数的影响：
```
autovacuum_vacuum_threshold = 50        #阈值
autovacuum_vacuum_scale_factor = 0.2    #比例因子

autovacuum_analyze_threshold = 50        #阈值
autovacuum_analyze_scale_factor = 0.2    #比例因子
```
每当死元组的数量超过时就会触发清理,公式为:
```
autovacuum_vacuum_threshold + pg_class.reltuples * autovacuum_vacuum_scale_factor
autovacuum_analyze_threshold + pg_class.reltuples * autovacuum_analyze_scale_factor
```

当满足上面的公式时,该表将被视为需要清理。该公式基本上表示在清理之前，高达20％的表可能是死元组(50行的阈值是为了防止非常频繁地清理微小的表)。默认的比例因子适用于中小型表，但对于非常大的表则没有那么多(在10GB表上,这大约是2GB的死元组,而在1TB表上则是~200GB)。
　
这是一个累积大量死元组并立即处理所有这些元素的例子,这将会严重影响数据库性能。根据前面提到的规则，解决方案是通过降低比例因子：
```
autovacuum_vacuum_scale_factor = 0.01
autovacuum_analyze_scale_factor = 0.01
```
这将限制表中的数据变化达到1%时触发autovacuum。

另一种解决方案是完全放弃比例因子，并仅使用阈值(建议使用)：
```
autovacuum_vacuum_scale_factor = 0
autovacuum_vacuum_threshold = 10000
autovacuum_analyze_scale_factor = 0
autovacuum_analyze_threshold = 10000
```
这将生成10000个死元组后触发清理。

这些参数可以在`postgresql.conf`中修改。同时，我们也可以根据不同表的数据量，根据各个表的delete和update频繁程度单独设置阈值：
```
ALTER TABLE tablename SET (autovacuum_vacuum_scale_factor = 0, autovacuum_analyze_scale_factor = 0,  autovacuum_vacuum_threshold = 10000, autovacuum_analyze_threshold = 10000);
```

### 手动Vacuum
虽然postgresql又自动清理死元组任务，但是对于某些表大多数情况下只有insert操作，几乎没有delete与update操作，所以很难甚至根本达不到达到autovacuum的阈值，那么我们也可以做一个cron任务，在后台空闲时进行手动vacuum操作：
```
vacuum analyze tablename;  //分析并统计可见元组与死元组，规划最佳查询计划
vacuum tablename;  // 清理死元组，释放磁盘空间
```
如果表只有insert，没有或者很少有delete与update操作，我们只需要进行vacumm analyze就可以了。

由于vacuum对CPU及IO的消耗是比较大的，所以我们一定要注意只有空闲时采取执行，并且不要频繁操作，每天一次即可。

