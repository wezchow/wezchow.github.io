---
layout:         post
title:          "PyMongo Aggregation"
date:           2015-03-08
categories:     Python MongoDB
tags:           PyMongo Python MongoDB
image:          /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

<br/>
最近一直在学点新玩意，基本都是围绕着 Python 这个语言，从爬虫（Scrapy），前后台框架 (AngularJS + REST Framework + DJango) 到数据库 (MongoDB) 统统过了一边。今天的进度落在 PyMongo 的 Aggregation (聚合) 功能上，发现中文的资料还是比较少，就勉为其难的翻译一下官方资料作为学习笔记吧。

[官方资料在这里](http://api.mongodb.org/python/current/examples/aggregation.html?highlight=aggregation)

# 实战聚合 (Aggregation)
在 MongoDB 中实现聚合功能有几种途径。下文中的例子就涵盖了通过全新的聚合框架 (new aggregation framework)，通过 map reduce 和通过 group 实现聚合操作的几种方法。

## 准备工作
在动手前，我们需要准备一些测试聚合功能的实验数据：

{% highlight python %}
>>> from pymongo import MongoClient
>>> db = MongoClient().aggregation_example
>>> db.things.insert({"x": 1, "tags": ["dog", "cat"]})
ObjectId('...')
>>> db.things.insert({"x": 2, "tags": ["cat"]})
ObjectId('...')
>>> db.things.insert({"x": 2, "tags": ["mouse", "cat", "dog"]})
ObjectId('...')
>>> db.things.insert({"x": 3, "tags": []})
ObjectId('...')
{% endhighlight %}

# 聚合框架 (Aggregation Framework)
这个例子展示了我们应该如何利用聚合框架中的 aggregate() 方法。我们将会实现一个简单的聚合操作，需要达到的目的是，计算集合内所有的 tags 数组中每种标签的出现次数。为了达到我们的目标，我们需要在聚合流水线 (pipeline) 中进行三步操作。第一，我们需要解绑 (unwind) tags 数组，接着创建标签与出现总数的集合，最后我们按标签出现总数进行降序排序。

由于 Python 的字典对象 (dictionaries) 并不能管理排序，我们应该在需要排序的时候 (eg "$sort") 使用 SON 对象或者 collections.OrderedDict 对象。

>注意：聚合框架需要 MongoDB 服务器版本 >= 2.1.0。而 PyMongo aggregate() 方法则需要PyMongo 版本 >= 2.3

{% highlight python %}
>>> from bson.son import SON
>>> db.things.aggregate([
...         {"$unwind": "$tags"},
...         {"$group": {"_id": "$tags", "count": {"$sum": 1}}},
...         {"$sort": SON([("count", -1), ("_id", -1)])}
...     ])
...
{u'ok': 1.0, u'result': [{u'count': 3, u'_id': u'cat'}, {u'count': 2, u'_id': u'dog'}, {u'count': 1, u'_id': u'mouse'}]}
{% endhighlight %}

不仅在简单的聚合操作上，聚合框架提供了数据的投射能力以便我们重新组织返回的数据。通过使用投射和聚合，我们可以添加计算后的字段，创建虚拟的子对象，还可以提取子字段作为返回结果的顶层数据。

>这里可以跳转到完整的 [MongoDB's aggregation framework 帮助文档](http://docs.mongodb.org/manual/applications/aggregation)

# Map/Reduce
实现聚合操作的另一个选择是使用 map reduce 框架。这里我们将定义 map 和 reduce 函数同样实现对集合中 tags 数组的标签进行统计。

我们的 map 函数只是针对数组中每一个标签，简单的弹射一个 (key, 1) 结构的键值对。

{% highlight python %}
>>> from bson.code import Code
>>> mapper = Code("""
...               function () {
...                 this.tags.forEach(function(z) {
...                   emit(z, 1);
...                 });
...               }
...               """)
{% endhighlight %}

Reduce 函数则负责累加计算所有已发射的键值，并把结果赋值给当前的键。

{% highlight python %}
>>> reducer = Code("""
...                function (key, values) {
...                  var total = 0;
...                  for (var i = 0; i < values.length; i++) {
...                    total += values[i];
...                  }
...                  return total;
...                }
...                """)
{% endhighlight %}

>注意：我们不能简单的返回 value.length，因为 reduce 函数可能在其他 reduce 过程中会被循环调用

最后我们使用 map_reduce() 方法并得到迭代后的结果集。

{% highlight python %}
>>> result = db.things.map_reduce(mapper, reducer, "myresults")
>>> for doc in result.find():
...   print doc
...
{u'_id': u'cat', u'value': 3.0}
{u'_id': u'dog', u'value': 2.0}
{u'_id': u'mouse', u'value': 1.0}
{% endhighlight %}

# 高级 Map/Reduce
PyMongo 的 API 支持所有 MongoDB map/reduce 引擎的功能。一个有意思的功能是通过对 map_reduce() 方法增加 full_response=True 参数，可以让我们在需要时获取更为详细的数据结果。和先前的区别在于，返回的结果会涵盖所有针对 map/reduce 操作的响应数据：

{% highlight python %}
>>> db.things.map_reduce(mapper, reducer, "myresults", full_response=True)
{u'counts': {u'input': 4, u'reduce': 2, u'emit': 6, u'output': 3}, u'timeMillis': ..., u'ok': ..., u'result': u'...'}
{% endhighlight %}

PyMongo 同样支持所有其他可选的 map/reduce 参数，在下面的例子中我们使用了 query 参数来限制需要被处理数据的范围。

{% highlight python %}
>>> result = db.things.map_reduce(mapper, reducer, "myresults", query={"x": {"$lt": 2}})
>>> for doc in result.find():
...   print doc
...
{u'_id': u'cat', u'value': 1.0}
{u'_id': u'dog', u'value': 1.0}
{% endhighlight %}

>这里可以跳转到完整的 [MongoDB's map reduce engine](http://www.mongodb.org/display/DOCS/MapReduce)

#Group
group() 方法提供了类似 SQL GROUP BY 的功能。相较于 map/reduce 更简单一些，你需要一个 group by 的键，一个聚合的初始值，和一个 reduce 函数。

>注意：不要在 Sharded MongoDB 集群上使用，用 aggregation 或者 map/reduce 代替 group()

这里我们做一个简单的 group 并计算 x 值出现的次数：

{% highlight python %}
>>> reducer = Code("""
...                function(obj, prev){
...                  prev.count++;
...                }
...                """)
...
>>> from bson.son import SON
>>> results = db.things.group(key={"x":1}, condition={}, initial={"count": 0}, reduce=reducer)
>>> for doc in results:
...   print doc
{u'count': 1.0, u'x': 1.0}
{u'count': 2.0, u'x': 2.0}
{u'count': 1.0, u'x': 3.0}
{% endhighlight %}

>这里可以跳转到完整的 [MongoDB's group method](http://www.mongodb.org/display/DOCS/Aggregation#Aggregation-Group)