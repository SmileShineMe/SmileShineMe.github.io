---
layout: post
title:  "Spark pairRDD基本操作"
date:   2025-01-06 09:00:00 +0800
categories: jekyll update
---
本文介绍Spark对键值对数据的基本操作，在Spark中键值对数据被称为pairRDD。
### 创建pairRDD
创建pairRDD的方式有很多，当需要把一个普通RDD类型转化为pairRDD类型时，map()函数是一个好的方法，下面的代码展示了如何将一个文本的每一行的第一个单词作为key，行数据作为value获取pairRDD的方式：
```
pairs = lines.map(lambda x: (x.split(" ")[0], x))
```
如果要从内存中已存在的二元组创建pairRDD,可用如下方式：
```
pairs = sc.parallelize({("key1", 1), ("key2, 3")})
```
### pairRDD的转化操作
#### 聚合操作
之前的文章中介绍过RDD的reduce操作可用来规约数据，针对pairRDD类型，有类似的函数--reduceByKey()，该方法会对数据集中的每个键进行规约操作，例如：
```
>>> pairs1 = sc.parallelize({("key1", 2), ("key2", 4), ("key1", 1), ("key3", 4)})
>>> pairs1.reduceByKey(lambda x, y: x + y).collect()
[('key1', 3), ('key2', 4), ('key3', 4)]
```
上述代码按key求和。

还可以使用reduceByKey进行文本单词统计，具体如下：
```
>>> rdd = sc.textFile("/home/lhao/Data/spark/log.txt")
>>> words = rdd.flatMap(lambda x: x.split(" "))
>>> words.map(lambda x: (x, 1)).reduceByKey(lambda x, y: x + y).collect()
[('this', 1), ('is', 2), ('test', 3), ('when', 1), ('read', 1), ('file!', 1), ('type', 3), ('match!', 2), ('pointer', 1), ('Goodbye!', 1), ('INFO:', 5), ('a', 1), ('ERROR:', 3), ('return', 2), ('error', 1), ('WARN:', 2), ('not', 3), ('exact', 3), ('double', 1), ('entry', 1), ('finished!', 1), ('NULL!!', 1), ('with', 1), ('error!', 1)]
```

Spark中最为常用的聚合函数是combineByKey()，combineByKey操作流程如下：

1. 如果是一个新元素，combineByKey会使用一个叫作createCombiner()的函数来创建那个键对应的累加器的初始值，这一过程一般在每个分区中第一次出现各个键时发生。
2. 如果这是一个在处理当前分区之前已经遇到过的键，则调用mergeValue()将该键的累加器对应的当前值与这个新值合并。
3. 由于各个分区独立处理，因此同一个键可以有多个累加器。如果有两个或更多的分区都有对应同一个键的累加器，就需要使用用户提供的mergeComniner()方法将各个分区的结果进行合并。

由以上描述可知，使用combineByKey方法需要提供三个函数：创建各个键的累加器初始值、分区内合并、分区合并。

举个例子，求各个键的值的和
```
nums.combineByKey((lambda x: (x, 1)),
                  (lambda x, y: (x[0] + y, x[1] + 1)),
                  (lambda x, y: (x[0] + y[0], x[1] + y[1])))
```
将上述代码应用到下列例子中：

|key|value|
|:---:|:---:|
|coffee|1|
|coffee|2|
|panda|3|


|key|value|
|:---:|:---:|
|coffee|9|

展示了两个分区的信息，我们来遍历这些分区元素:
第一个分区中，遇到("coffee", 1)这个元素时，由于时第一次遇到键"coffee"，调用lambda x: (x, 1)生成新的累加器初始值：accumulators["coffee"] = (1, 1), 遇到("coffee", 2)时，由于第一个分区之前已存在"coffee"这个key，因此调用lambda x, y: (x[0] + y, x[1] + 1)进行分区内合并：
accumulators["coffee"] = (3, 2), 对于第三个元素会生成accumulators["panda"] = (3, 1).

对于第二个分区，因"coffee"是首次出现，因此调用lambda x: (x, 1)生成新的累加器初始值：accumulators["coffee"] = (9, 1)

最后合并两个分区，得到：
```
accumulators["coffee"] = (12, 3)
accumulators["panda"] = (3, 1)
```
#### 数据分组
数据分组常用操作是groupByKey(),用法如下：
```
>>> pair = sc.parallelize({("key1", 2), ("key2", 1), ("key1", 5), ("key2", 5), ("key3", 1)})
>>> pair.groupByKey().collect()
[('key1', <pyspark.resultiterable.ResultIterable object at 0x7f29424f9cc0>), ('key2', <pyspark.resultiterable.ResultIterable object at 0x7f29424f9198>), ('key3', <pyspark.resultiterable.ResultIterable object at 0x7f2942568d68>)]
```
#### 连接
用法如下：
```
>>> pair1 = sc.parallelize({("key1", 1), ("key2", 2), ("key3", 3)})
>>> pair2 = sc.parallelize({("key2", 3), ("key3", 3), ("key4", 1)})
>>> pair1.join(pair2).collect()
[('key3', (3, 3)), ('key2', (2, 3))]
>>> pair1.leftOuterJoin(pair2).collect()
[('key3', (3, 3)), ('key1', (1, None)), ('key2', (2, 3))]
>>> pair1.rightOuterJoin(pair2).collect()
[('key3', (3, 3)), ('key2', (2, 3)), ('key4', (None, 1))]
```