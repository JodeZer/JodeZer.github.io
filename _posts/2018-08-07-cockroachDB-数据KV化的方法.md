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


key | val
---- | ---
/test/10/floatVal | 4.5
/test/10/stringVal |  "hello"


在这个标书中，我们使用`/test/`来代替标识表ID，用`/floatVal`和`/stringVal`代替标识列ID（每个行都有一个表内唯一的ID）。需要注意的是，主键在我们的编码中紧跟表ID。这在CockroachDB的SQL实现中，是索引扫描的基础。


假设表的元数据长这样：

key | val 
---- | ---
`test Table ID` | 1000
`key Column ID` | 1
`floatVal Column ID` | 2
`stringVal Column ID` | 3

那么上述行数据，就是这样的：

Key | Value
---- | ---
`/1000/10/2` | `4.5`
`/1000/10/3` |  `"hello"`

文章剩下的部分，都会使用这样的表述方式。

[你也许认为这些key的公共前缀设计`/1000/10`在key中重复出现是浪费空间（注：比如用一个更短的key的字典表来替代），但底层存储RockdsDB，使用前缀压缩来消除了这样的开销。（注：所以在应用层不需要在这个上面重复工作）]

机智的读者会发现，在这样的kv对中，主键列的值已经包含在key中了，所以对主键列，CockroachDB就省略了。

具体到某一行数据，他的每列数据总是存储在相邻的位置，因为他们的key前缀都是主键列，是一样的（在CockroachDB中，kv数据是单调序列，这样的特性是天然的。）这样，取某一行的某几列数据的时候，需要扫描那一列的所有key，CockroachDB内部就是这样做的。

假设查询：

`SELECT * FROM test WHERE key = 10`

会被翻译成：

`Scan(/test/10/, /test/10/Ω)`

`Ω`符号标识最后一个可能key的后缀。查询执行引擎之后会通过这些key，构造出被查询的列数据。


#### 空值列（介绍Sentinel Key）

一个小插曲：没有被标记为`NOT NULL`的列值可以为`NULL`。CockroachDB不存储`NULL`值，使用key的缺失来标识这一列是空值。这样又有一个奇怪的小事情，如果一行数据，除了主键列，全部为空，那么这一行数据在CockroachDB中就不存在了（注：因为主键列的数据用key来标识了）。为了搞定这样的情况，CockroachDB会给每一行写一个哨兵key（Sentinel Key）,这个哨兵key只包含表ID，主键列的值，没有任何其他非主键列的列ID。比如对于行数据`<10,4.5,"hello">`，哨兵key就是`/test/10`。

#### Secondary Index

那么Secondary Index又咋么搞咧？
比如：
`CREATE INDEX foo ON test (stringVal)`
这个语句会在stringVal列上创建一个非唯一的secondary index。同样的，索引kv化时，key以表ID作为前缀。但我们希望把索引数据跟行数据分开。所以每个索引都有一个表内唯一的ID，包括主键列的索引（注：所以上面举的例子需要重新补充一下，但是无伤大雅）：
/tableId/indexId/indexColumns[/columnId]

那么例子里的行数据就会变成

Key | Value
---| ---
`/test/primary/10` | `∅`
`/test/primary/10/floatVal` | `4.5`
`/test/primary/10/stringVal` | `"hello"`

现在我们增加了一个索引数据：
Key | Value
---| ---
`/test/foo/"hello"/10` | `∅`

key中不仅有该索引的ID和值，在最后还附加上了主键列的值，原文描述是在结尾附加{ {主键列的值} - {索引列的值} }。这里原文罗里吧嗦了一堆为什么要加上主键列。我个人理解，跟mysql的二级索引原理一样，是通过二级索引查找到主索引，然后查找行数据。
（注：这里我有个疑问就是，不加上主键列，把主键列作为value，照样可以起到搜索的作用。莫非是RocksDB层，只扫描KEY和查询值是两步操作？这样可以更高效）
假设现在加一行`<4,NULL,"hello">`进表，那么这个表的所有数据就会变成：

Key | Value
--- | ---
/test/primary/4 | Ø
/test/primary/4/stringVal |	"hello”
/test/primary/10	| Ø
/test/primary/10/floatVal	| 4.5
/test/primary/10/stringVal	| "hello”
/test/foo/"hello”/4	| Ø
/test/foo/"hello”/10|	Ø

假设执行:
`SELECT key FROM test WHERE stringVal = "hello"`

执行计划器会发现在stringVal这一列上有索引，然后就会把查询翻译成：
`Scan(/test/foo/"hello"/, /test/foo/"hello"/Ω)`
然后就会找到下列的key：

Key	| Value
--- | ---
/test/foo/”hello”/4	| Ø
/test/foo/”hello”/10 |	Ø
然后通过附在最后的主键（前缀），去搜索该有的数据。

最后来看唯一索引怎么存储。假设创建一个唯一索引`uniqueFoo`:
`CREATE UNIQUE INDEX uniqueFoo ON test(stringVal)`

不同于普通索引，唯一索引的key就只包含列数据了（不要主键列了），主键列的值这回被放到了value中。仍然是差运算，{主键列的值} - {索引列的值}。这个索引的key就是这样的：

Key	| Value
--- | ---
/test/uniqueFoo/"hello”	| /4
/test/uniqueFoo/"hello”	| /10

（注：显然这个表当前就没法建唯一索引）

我们在写入的时候用ConditionalPut来保证这个索引的唯一性。
（注：这里我的理解是，跟普通索引的差别是不加主键列来保证唯一性，那么就不能用前缀来保证唯一性吗？类似ConditionalPrefixPut（Prefix, Key,Value）这样的语义？）

#### 原作者注
这种在KV数据上实现SQL的方法不是CockroachDB独创的。MysqlInnoDB， Sqlite4 一级其他数据库都有这样的设计。

#### 译者注
关于文中，对二级索引，普通索引加入主键列，唯一索引将主键列放入value中的做法，似乎有待详细了解RocksDB的更多特性来理解？

