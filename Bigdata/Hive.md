# Hive

Hive 是一种运行在Hadoop平台之上的类SQL语句。

## 表

Hive表中的数据存在HDFS目录中，但表的定义存储在一个名叫HCatalog的关系型数据库中。

- Hive 表中的数据和表的定义是松耦合的，如果将表定义为外部表，那么删除表定义和删除底层数据是相互独立的，不会像关系型数据库中，删除表后，也会同时删除存储的数据。
- Hive 中的单个数据集可以有多个表定义。
- HIve 表中的底层数据可以以多种格式存储。

实际数据和模式是分离的，也是Hadoop超越关系系统的重要价值主张之一。Hive模式只是一种元数据映射，这使得理解标准SQL的人和应用程序很容易查看底层数据。是一层对数据分析人员友好的接口。

### 创建表

语法与SQL非常相似。

```sql
CREATE EXTERNAL TABLE retail.customer (
    fname       STRING,
    lname       STRING,
    address     STRUCT <HOUSE:STRING, CITY:STRING, ZIPCODE: INT, STATE:STRING, COUNTRY:STRING>,
    -- 利用hive中提供的复合数据结构定义的类型address
    active      BOOLEAN,
    created     DATE)
COMMENT "customer master record table"
LOCATION '/usr/demo/customers';
```

命名表名时如果不指定数据库，那么会存在默认的default数据库中创建该表。


### 列出所有表

```sql
SHOW TABLES IN retail;
```
如果数据库中存有很多张表，也可以使用通配符来搜索特定的表。

### 内部外部表

Hive的表的类型分为内部表和外部表。 表的类型决定了Hive如何加载、存储和控制数据。

- 外部表

在create table语句中使用external关键字创建外部表。这是hadoop所有生产部署中推荐的表类型。因为大多数情况下，底层数据会被用于多个用例。
适合使用外部表的情况：
    
    - 想删除表定义，但不删除底层数据
    - 数据存储在文件系统中而不是在HDFS上，
    - 希望使用自定义位置存储表数据
    - 不准备基于另一个表来创建表
    - 数据被多个处理引擎访问（既希望使用Hive来读取表，有希望在Spark程序中使用该表)
    - 希望在同一数据集中创建多个表的定义。删除一个定义的同时不会影响到其他的定义。

- 内部表

内部表是指数据由hive管理的表，当删除一个内部表的时候，底层的数据也会被删除。这不是很常用。
适合内部表的情况：

    - 数据是临时存储的。
    - 访问数据的唯一方式是通过Hive，而且需要用Hive来完全管理表和数据的生命周期。
