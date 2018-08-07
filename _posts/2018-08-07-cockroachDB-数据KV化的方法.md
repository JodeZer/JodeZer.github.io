---
layout: post
title:  CockroachDB-数据KV化的方法
date: 2018-08-07 17:25:37 +0800
comments: false
---


# 简介
CockroachDB是一款开源的newSql数据库，其底层存储使用的是RocksDB，本文翻译了官方博客上，介绍如何将类mysql的行数据转化为KV的处理方法。[博客原文](https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/)

# Content

一个SQL表有多行，其中每一行又有多个列，每一列都有一个类型（布尔，整型，浮点，字符串，字节等）。每个表也有会关联的索引来支持高校的行数据检索。那么如何将这样的数据最终映射到键值对（string:string）这样的模型呢？

首先，CockroachDB的内部KV API支持很多操作，在此只介绍两个：
- `ConditionalPut(key, value, expected-value)` - 有条件地写入值，当该key的值与期望的值相同。
- `Scan(start-key, end-key)` - 左闭右开地检索在这个范围内的key

## Key的编码

首先要有一种编码方式把SQL表的行信息编码进key当中。比如给一组数据`<1,2.3,"four">`，我们可以把这个数据编码成`/1/2.3/"four"`

为了可读，我们用斜杠来作为一个虚拟的分隔符。数据的编码实现方法在这里不讨论，处于方便，这篇文章只讨论这些东西的特点，不会涉及实现细节。