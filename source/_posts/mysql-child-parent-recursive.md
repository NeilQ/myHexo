title: MySql由child找parent
date: 2016-08-18 14:45:27
tags:
categories:
- DataBase

---

记录sql:

```sql
SELECT s.*, @code :=
(SELECT  ParentCode FROM mytable
WHERE Code = @code) AS tmp
FROM  (SELECT  @code := 10) vars
STRAIGHT_JOIN mytable s
WHERE @code IS NOT NULL;
```
