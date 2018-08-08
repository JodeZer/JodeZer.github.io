---
layout: post
title:  CockroachDB-数据KV化的方法
date: 2018-08-07 17:25:37 +0800
comments: false
---


## 简介
CockroachDB是一款开源的newSql数据库，其底层存储使用的是RocksDB，本文翻译了官方博客上，介绍如何将类mysql的行数据转化为KV的处理方法。[博客原文](https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/)

## Content

一个SQL表有多行，其中每一行又有多个列，每一列都有一个类型（布尔，整型，浮点，字符串，字节等）。每个表也有会关联的索引来支持高校的行数据检索。那么如何将这样的数据最终映射到键值对（string:string）这样的模型呢？

首先，CockroachDB的内部KV API支持很多操作，在此只介绍两个：
- `ConditionalPut(key, value, expected-value)` - 有条件地写入值，当该key的值与期望的值相同。
- `Scan(start-key, end-key)` - 左闭右开地检索在这个范围内的key

#### Key的编码

首先要有一种编码方式把SQL表的行信息编码进key当中。比如给一组数据`<1,2.3,"four">`，我们可以把这个数据编码成`/1/2.3/"four"`

为了可读，我们用斜杠来作为一个虚拟的分隔符。数据的编码实现方法在这里不讨论。（然后介绍一个奇怪的不可用的编码方法，可能是想举个反例吧，就无视好了）。

在讨论编码方案之前，我们先看一下要编码的SQL表数据。在CockroachDB中，每个表在创建的时候会被分配一个唯一的64位整型ID。这个表ID用来将这个表的所有KV数据联系在一起。现在来考虑下列的表和数据：
```
CREATE TABLE test (
	key 		INT PRIMARY KEY,
	floatVal  FLOAT,
	stringVal STRING
)

INSERT INTO test VALUES(10, 4.5, "hello")
```

每个CockroachDB都必须有一个主键，这个主键可能有一到多列组成；在上述的`test`表中，主键只有单列。CockroachDB将每个非主键列的数据存储在不同的key中，以主键key值为前缀，以列的名字为后缀。比如行`<10,4.5,"hello">`在`test`表中的存储会是：

name | age
---- | ---
`/test/10/floatVal` | `4.5`
`/test/10/stringVal` |  `"hello"`


在这个标书中，我们使用`/test/`来代替标识表ID，用`/floatVal`和`/stringVal`代替标识列ID（每个行都有一个表内唯一的ID）。需要注意的是，主键在我们的编码中紧跟表ID。这在CockroachDB的SQL实现中，是索引扫描的基础。








