---
title: Oracle中的exists和in
date: 2016-09-19
categories:
  - 技术
tags: 
  - Oracle
---

## 写在前

今天有一个业务，前台是一个供用户选择条件进行查询的功能。选择完毕后点击查询，前台使用get方式发送打包好的参数列表到后台（Controller），持久层拿到参数列表后拼装到SQL中，使用jdbc连接oracle拿数据。

## 问题

当条件过大（据说有7000多个数据），出现第一个问题，413 FULL HEAD ERROR。很明显是数据量过大，而使用的又是get方式传输，get请求的数据会附在URL之后（就是把数据放置在HTTP协议头中），以?分割URL和传输数据，参数之间以&相连。如：login.action?name=cym&password=idontknow。如果数据是英文字母/数字，原样发送，如果是空格，转换为+，如果是中文/其他字符，则直接把字符串用BASE64加密。GET方式提交的数据依据浏览器而定，IE环境下的URL长度限制为2083，理论上POST没有限制，可传较大量的数据。

第二个问题出现在SQL，拼接字符串中使用的where...in..，JDBC会抛出“java.sql.SQLException: ORA-01795: 列表中的最大表达式数为 1000”这个异常。没想到oracle还有这种限制，据说11g后不存在这个限制。

## 解决方案

### 一、使用临时表

```sql
SELECT * FROM t WHERE id IN (SELECT id FROM temp)
```

建立一个表来存储传入参数，取数据的时候先从临时表中遍历取参数，然后再将取出的参数拿去作为in条件

### 二、使用exists

```sql
SELECT * FROM a WHERE EXISTS( SELECT 1 FROM b WHERE a.employee_id=b.employee_id)
```

## 对比

- IN比 EXISTS 的可读性更好
- EXISTS 比IN 的表达性更好（更适合复杂的语句）

二者之间性能没有差异（但对于某些数据库来说性能差异会非常大）