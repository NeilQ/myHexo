title: mysql5.6 升级到 mysql5.7遇到的问题
date: 2016-11-18 15:41:11
tags:
- MySql
categories:
- DataBase
---

最近从MySql5.6升级到5.7版本，踩到一些吭，在此记录。

## sql_mode=ONLY_FULL_GROUP_BY
项目中某些sql报错
```
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP
BY clause and contains nonaggregated column 'mydb.t.address' which
is not functionally dependent on columns in GROUP BY clause; this
is incompatible with sql_mode=only_full_group_by
```

原因: 某些sql不规范，select的字段没有全部包含在group by子句中，相关mysql手册请参考[这里](http://dev.mysql.com/doc/refman/5.7/en/group-by-handling.html)和[这里](http://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by)

临时解决方案: 将sql-model中的only_full_group_by移除。

## sql_mode=STRICT_TRANS_TABLES
某些字段已经设置为text类型，但插入数据时还是报错
```
Data too long for column 'Content' at row 1
```

原因: 与mysql对表列数据长度限制有关，详细描述参考[这里](http://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html)

临时解决方案: 将sql-mode中的STRICT_TRANS_TABLES移除

## 如何修改sql-mode
MySql5.7默认的sql-mode为"ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"，找到 my.ini (Windows)或者my.cnf (linux)文件，添加
```
[mysqld]
sql-mode="NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
```
将移除上述两种sql-mode。
