---
layout: post
title: "Tablesaw简介"
author: "Tangoo"
categories: Bigdata
tags: [tablesaw,hive]
---

最近一直在学习大数据相关的内容，技术坑很多，水很深。偶然间在一位博主的博客中看到了「Tablesaw」这款开源工具，号称「MAKING BIGDATA SAMLL AGAIN」，觉得很神奇，遂前去一探究竟。

## Tablesaw简介
引用官网的一段介绍：
> Tablesaw is a high-performance, in-memory data table in Java.
>
> Tablesaw's design is driven by two ideas: 
> - First, few people need distributed analytics. On a single server, Tablesaw lets you work interactively with a 2,000,000,000 row table. (I plan to raise that ceiling, btw.) 
> - Second, it should be super easy to use: To that end I happily steal ideas from everything from spreadsheets to specialized column stores like KDB.

我们来总结下，Tablesaw设计初衷有以下两条：
1. 作者认为很少有人真的需要进行分布式计算分析，在单台服务器上「Tablesaw」能支持最多20亿条数据，似乎在大多数情况下都能满足用户的计算需求；
2. 大数据工具应该简单易用，费大把劲去搭建大数据集群估计已经吓退一拨人。
从本质上而言，我们发现「Tablesaw」就是一款基于内存，使用纯java实现的big spreadsheet。嗯，我相信你一定使用过`Excel`。

## Tablesaw主要功能
既然Tablesaw宣称自己简单好用，那到底支持哪些功能呢，笔者新建了一个Maven工程玩了一把，貌似有以下功能：
> - Import data from RDBMS and CSV files, local or remote (http, S3, etc.)
> - Add and remove columns
> - Sort
> - Filter
> - Group
> - Map and reduce operations
> - Descriptive stats (mean, min, max, median, etc.)
> - Plotting for exploratory data analysis and model checking
> - Store tables in a very-fast, compressed columnar storage format

在试用过程中有几点事项说明下：
1. 在数据量不大的情况下，执行简单的查询操作速度很快，不像在Hive中做一个非常简单的查询都会转化成一个MapReduce作业在Hadoop集群上跑几十秒时间(当然你可以通过`set hive.fetch.task.conversion=more`，使得简单的查询不会被转化成MapReduce作业)；
2. Tablesaw支持的功能还非常有限，对于复杂的组合查询会很吃力，甚至可能都做不到，Tablesaw作者在博客中强调未来会支持更多的列类型，支持更多的操作，同时把常用的机器学习算法的实现也引入进来，对于未来的设想我们拭目以待；
3. 在借助Tablesaw做数据展示的时候，使用的是XCharts。

## Tablesaw适用场景
简单介绍了Tablesaw的主要功能以及未来的设想，其实我更关注的是如何与我们现有的应用场景进行结合，或者说Tablesaw在哪些场景下会比较适用，大致能想到的有以下两个领域：
1. 作为缓存器，对于热点数据我们从持久化的关系型数据库读到内存之后，可以马上对外提供计算查询的API；
2. 对于数据分析师，数据科学家等以实验论证形式展开工作的一类人，对他们来讲有一款简单易用的大数据工具绝对是福音。