---
title: 如何在impala-shell之外使用特定用户set变量
date: 2020-05-20
categories:
  - 技术
tags: 
  - PHP 
  - impala
---

## 版本信息

- impalad 2.5.0-cdh5.7.*

## 问题

通过查阅[官网](https://docs.cloudera.com/documentation/enterprise/5-7-x/topics/impala_set.html#set)可以知道，用户特定变量无法在除``impala-shell``外使用，这对使用odbc、jdbc或thrift的客户端来说，SQL维护会造成很大困扰 —— 在长SQL中改变一个或多个条件的值

<!-- more -->

在impala-shell中，可以通过``set``语句或``--var``参数设置会话上下文中的变量，如下

### 使用set

```bash
[localhost:21000] > set var:table_name=production_table;
[localhost:21000] > set var:cutoff=3;
[localhost:21000] > select s from ${var:table_name} order by s limit ${var:cutoff};
Query: select s from production_table order by s limit 3
```

### 使用--var

```bash
$ impala-shell --var=table_name=staging_table --var=cutoff=2
[localhost:21000] > select s from ${var:table_name} order by s limit ${var:cutoff};
Query: select s from staging_table order by s limit 2
```

## 模拟impala-shell：通过程序替换

```sql
set var:table_name=production_table;
set var:cutoff=3;
select s from ${var:table_name} order by s limit ${var:cutoff};
```

有以上sql，

1. 通过正则表达式抽出``set语句``
2. 替换到由``${}``包裹的变量名
3. 使用odbc/jdbc/thrift执行替换变量后的sql
