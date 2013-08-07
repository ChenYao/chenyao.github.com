---
layout: post
title: "通过sqlplus批量执行SQL文件"
description: ""
category: 
tags: [Oracle, sqlplus, 批量执行]
---
{% include JB/setup %}

Oracle数据库中，可以在sqlplus中通过'@'操作符打开并执行文件的内容。

比如文件c:\a.sql的内容如下：

a.sql:

    select 1 from dual;
  

通过sqlplus输入如下：
SQL >@c:\a.sql

这条语句就会被执行。这样多条语句就可以写在同一个文件中一次执行，实际上这个方法对简单的sql来说用处并不大，@操作符更多的用在
批量创建存储过程或者函数这类以单独文件形式存在的数据库对象。这种情况下可以使用'@@'来执行文件中指向的文件。
比如，有3个存储过程的创建文件分别为：

	c:\sp\a.prc	
	c:\sp\b.prc	
	c:\sp\c.prc

这样可以建立一个文件c:\all.sql来“引用”这些存储过程如下：

all.sql:

	@@c:\sp\a.prc
	@@c:\sp\b.prc
	@@c:\sp\c.prc
  
通过sqlplus输入如下：

SQL >@c:\all.sql  

则可以同时创建这些单独的存储过程文件。
