```
---
layout:     note
title:      "PostgreSQL 使用笔记"
subtitle:   " \"PostgreSQL 使用笔记\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- PostgreSQL
---
```



## PostgreSQL 使用笔记

### 注意事项

PostgreSQL在同个`schema`中不允许同名的索引、视图、序列，具体可见`pg_catalog.pg_class.pg_class_relname_nsp_index`约定



### 语法

#### 字段case

```
select 
  case type
   when 1 then ""
   when 2 then ""
   else ""
  end
from a
```



#### 联表修改

```
update t set
t.a = tmp.a
from (select * from .....) tmp
where t.id = tmp.t_id
```



#### 按唯一索引(主键)插入或更新

```sql
insert into table (a, b, c)
values (), (), ....
on conflict (a) do update
set b = excluded.b, c = excluded.c
```



#### 取差集

```sql
select a from t_a 
except
select a from t_b;
```





### 函数区别

| MySQL                                         | PostgreSQL                               |
| --------------------------------------------- | ---------------------------------------- |
| `group_concat(distinct field separator  ',')` | `array_to_string(array_agg(field), ',')` |
|                                               |                                          |
|                                               |                                          |





### 错误

- 连接数超过限制，导致客户端无法连接

具体报错：`[53300] FATAL: remaining connection slots are reserved for non-replication superuser connections.  `   

解决方法：

```
修改配置增大最大连接数 max_connections
```

```
设置合理的连接池超时时间
```

```
show max_connections;  查看最大连接数
select * from pg_stat_activity; 查看当前连接数并记录将要结束的连接数PID。
SELECT pg_terminate_backend([$PID]) FROM pg_stat_activity;  结束连接数进程。
```





### pgbench

`pgbench`是`PostgreSQL`自带的基准测试工具，它可以模拟多个客户端同时执行事务并测量系统性能。`pgbench`不仅可以测试数据库服务器的吞吐量、响应时间和并发性能等指标，还可以用于模拟负载以及生成随机数据并将其插入到表中。
