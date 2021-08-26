---
layout: post
title: "Hadoop Hive入门"
author: "Tangoo"
categories: Bigdata
tags: [hadoop,hive]
---


在「聊聊大数据平台的典型应用场景」这篇文章中介绍了团队目前搭建大数据平台的实践，以及如何在实际生产环境中应用我们的大数据平台也就是寻找典型的应用场景。至于Hadoop生态系统中的Hive工具则是点到为止，并没有展开详细讨论，在这边文章中我们来入门Hive，了解Hive工具的作用，执行原理，数据类型以及数据模型。

引用官网的一段说明：
> The Apache Hive ™ data warehouse software facilitates reading, writing, and managing large datasets residing in distributed storage using SQL. Structure can be projected onto data already in storage. A command line tool and JDBC driver are provided to connect users to Hive.

Hive本质上是一个数据仓库，但不存储数据(只存储元数据)，用户可以借助Hive使用sql对存储在分布式文件系统中的大数据集进行读写。

## 概述
知悉Hive是一个数据仓库的本质之后，我们需要简单了解下什么是数据仓库：
> 数据仓库是面向主题的、集成的、汇总的、大容量的数据集合，主要提供企业决策分析之使用。

了解完以上这些基础信息之后，回到Hive这款开源工具本身，Hive建立在Hadoop之上，两者之间的依赖关系如下图1所示。
<figure>
   <img src='{{ "/img/2018/hive-arch.png" | absolute_url }}' />
   <figcaption>图1：THE ARCHITECTURE OF HIVE WITH HADOOP.</figcaption>
</figure>￼

结合上图对Hive与Hadoop的直观了解，做到心中有个大致的印象，现在来讲清楚四个我认为非常关键的内容：
1. Hive既然不保存实际数据，那数据存放在何处；
2. 用户向Hive提交查询请求时，Hive会如何处理；
3. Hive的数据类型；
4. Hive的数据模型；

## Hive数据存储
从图1可知，在Hive的体系结构中，我们只看到了Hive对元数据信息进行管理，而并没有保存实际的数据。根据Hive的官方定义，Hive显然把真正的表数据文件存放在了Hadoop的HDFS中，而在其内建的derby或外部的Mysql数据库中保存了表的元数据信息。

这里说到了Hive的元数据信息，关于Hive元数据需要了解以下两点：
1. Hive将元数据(metadata)存储在数据库中，支持Derby(Hive自带，若不指定其他关系型数据库，则Hive使用默认的derby数据库)、Mysql和Oracle等数据库；
2. Hive中的元数据包括表的名字，列字段及类型，表的类型(内部表，分区表，外部表和桶表)以及表数据文件在HDFS上的存储目录。

假设现在我们在Hive中创建一张普通的内部表，执行的hql语句如下：
```roomsql
create table test (
    sid int, 
    name string, 
    age int
);
```
那么Hive内部数据存储时到底发生了什么，Hive首先会在元数据库中记录下表`test`的信息(表的名称，字段，字段类型以及在HDFS中的存储路径)；其次在HDFS相应的目录中会生成一份数据文件用来存放`test`表中具体的数据内容，数据文件一般存放在`/user/hive/warehouse/test/data.txt`(存放路径也可以在创建表的时候进行指定)。从这里可以看出Hive中的一张表对应HDFS中的一个目录，在该目录下存放表中的数据内容文件。

## Hive执行过程
Hive执行过程可以用下图2进行说明，我们以一条查询语句为例：
```roomsql
select name,age from test where age>=25;
```
<figure>
   <img src='{{ "/img/2018/hive-exec.png" | absolute_url }}' />
   <figcaption>图2：EXECUTION PROCESS OF HIVE WITH HADOOP.</figcaption>
</figure>￼

1. HQL语句首先接受词法分析，检查是否有词法上的错误；
2. 词法检查通过之后，在编译时生成hql的执行计划，可以通过命令生成执行计划：`explain plan for select name,age from test where age>=25; `，然后通过该命令查看执行计划：`select * from table(dbms_xplan.display); `;
3. 接下来Hive会对执行计划进行优化，比如我们查询的表是一张分区表，根据条件优化之后可能只会取某几个分区中的数据，而无需扫描该表对应HDFS目录下的所有数据文件；
4. 等到一切条件准备成熟之后，执行器会把执行计划发往MapReduce框架中的Job Tracker，执行批量查询。

一般来说所有的查询请求最终都会转化成一个MapReduce任务在集群上执行，但存在例外情况，比如：`select * from test`，该查询并不会转化成MapReduce任务，因为只要把该表对应HDFS目录下的所有数据文件全部取出即可。

## Hive数据类型
Hive既然是一个数据仓库，本质上就是一个数据库，库中有表，表里面有定义好的列，现在我们来简单介绍列的数据类型，根据Hive官网介绍，Hive数据类型大体上可以分成以下几个大类：
> 1. Numeric Type
> 2. Date/Time Types
> 3. String Types
> 4. Misc Types
> 5. Complex Types

以上1-4种数据常用简单，就不再赘述，使用时自行参考官网，对`5.Complex Types`稍微进行解释说明，根据官网的说明`Complex Types`有以下几类：
```html
arrays: ARRAY<data_type> (Note: negative values and non-constant expressions are allowed as of Hive 0.14.)
maps: MAP<primitive_type, data_type> (Note: negative values and non-constant expressions are allowed as of Hive 0.14.)
structs: STRUCT<col_name : data_type [COMMENT col_comment], ...>
union: UNIONTYPE<data_type, data_type, ...> (Note: Only available starting with Hive 0.7.0.)
```

比如我们在使用`ARRAY`(保存同一种数据类型的数据)这种复杂类型创建表的时候就可以像下面这样写：
```roomsql
create table student_tbl_(
    sid int, 
    name string, 
    credit<float>
);
```
credit是一个保存浮点数的数组，用来描述每位学生的各科绩分。

## Hive数据模型
讨论完表中的字段类型，现在来看看表的分类，大致有以下几种：
> - 内部表
> - 分区表
> - 外部表
> - 桶表
> - 视图

每种数据模型在官网都有详尽介绍，在这里讲解一个我们在实践过程中使用比较多的一种数据模型：***分区表***，以表中的某个字段进行分区，比如我们创建一张学生表如下，带上出生日期作为分区字段：
```roomsql
create table student_tbl_ (
    sid int, 
    name string
) partitioned by (birthdate DATE)；
```

即该表中包含了三个字段分别是学生编号，姓名和出生日期，同时把出生日期作为分区字段，则表中的数据内容对应与HDFS中的存储结构可能是这样的：
```text
/user/hive/warehouse/student_tbl/birthdate=199501/data0.txt
/user/hive/warehouse/student_tbl/birthdate=199502/data0.txt
```
总结一下：在Hive中，表中的一个Partition对应与HDFS表名下的一个目录，属于该Partition的数据都会被存储在该目录中。