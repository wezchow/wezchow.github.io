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
TODO