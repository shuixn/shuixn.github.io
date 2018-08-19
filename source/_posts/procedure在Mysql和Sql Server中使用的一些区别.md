---
title: procedure在Mysql和Sql Server中使用的一些区别
date: 2015-12-17
categories:
  - 技术
tags: 
  - 数据库
---

## 环境
    
- MySQL 5.6.20

## 写在前

近日，一个好友@随风飘散，问我一个关于procedure在MySQL中使用出现的一些问题。在解决后做一个简单的总结：

## 正文

下面这段代码来自SQL Server，意思是创建一个procedure， 查询表test_user所有的记录。

```sql

CREATE PROCEDURE getUsers
AS
BEGIN
SELECT * FROM test_user
END

EXECUTE    getUsers
```

将这段代码原封不动的复制到MySQL中执行

![](/images/20151217113258531.jpg)

可以看出，很明显是语法不正确出现的问题。经查阅相关文档，得出以下代码段

```sql
DROP PROCEDURE IF EXISTS getUsers;
DELIMITER //

CREATE PROCEDURE getUsers()
BEGIN

SELECT * FROM test_user;

END
//
DELIMITER ;

CALL  getUsers();
```

执行结果如下：

![](/images/20151217120502176.jpg)

可以看出，procedure 在 **sql server** 和 **mysql** 中使用的差别还是很大的， 有几个重要的区别：


1. 使用分隔符包含存储过程
2. 不再需要AS
3. 使用CALL代替EXECUTE

解决了这个问题，甚是感觉到自己实践不足，遇到的问题太少，希望自己能遇到更多的问题，提高自己的同时，帮助更多的人解决问题。