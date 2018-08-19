---
title: C Sharp连接Access数据库出现的问题
date: 2014-03-29
categories:
  - 技术
tags: 
  - C Sharp
  - 问题排查
---

这两天在做数据库，发现一个很多网友都头疼的问题，就是C#连接Access数据库的时候显示“不可识别的数据库类型”。

很多人在做Access数据库的时候总是默认保存它的后缀名“.accedb”，然后用Provider=Microsoft.Jet.OLEDB.4.0;Data Source=...去连接Access数据库。

首先第一个问题就是C#是不识别后缀“.accedb”的，因此只能将该格式转换为C#识别的“.mdb”。

然后很多人会发现还是提示“不可识别的数据库类型”！其实原因就在于“Provider=Microsoft.Jet.OLEDB.4.0;Data Source=...”。因为那个 Microsoft.Jet.OLEDB.4.0， 是针对低版本的 Access 使用的。

只需要把

```
Microsoft.Jet.OLEDB.4.0
```

改为

```
Microsoft.ACE.OLEDB.12.0
```

数据源就可以正常的输出啦！