---
layout: post
title:  "MongoDB注入攻击案例分析"
date:   2018-09-03 16:51:53
categories: 网络安全
tags: SQL注入
---

* content
{:toc}

在开始我们的MongoDB“注入之旅”之前，我们需要先知道和其他数据库相比，为什么我们更愿意选MongoDB——因为MongoDB并不是SQL作为查询语句，所以人们可能会以为这样的数据库难以进行注入攻击？然而事实上并非如此。

FreeBuf百科：关于MongoDB
简单的说，MongoDB是个开源的NoSql数据库，其通过类似于JSON格式的数据存储，这使得它的结构就变得非常自由。通过MongoDB的查询语句就可以查询具体内容。 

为什么使用MongoDB？
其实多半只是因为MongoDB可以快速查找出结果，它大概可以达到10亿/秒。当然MongoDB很流行的另外一个原因是在很多应用场景下，关系型数据库是不适合的。例如，使用到非结构化，半自动化和多种状态的数据的应用，或者对数据可扩展性要求高的。

如果你想测试下你的开源程序，可以在以下网站中测试：

http://blog.securelayer7.net/securelayer7-gratis-pentest-summer-2016/

攻防案例
好了，我们来看一个注入案例

第一个php例子，页面主要实现通过变量id获取到该id的username和password，

页面和源代码截图如下：

 图片26.png图片27.png  

由上图源代码可以知道，后台数据库的名字是security，集合名是users。u_id 是通过GET请求传到后台，然后传入一个数组变量中。然后进入MongoDB的查询。接下来，我们试试通过数组传入运算符号。

图片28.png 

看看结果，返回了数据库中的所有内容。我们分析下我们传入的数据：


http://localhost/mongo/show.php?u_id[$ne]=2
传入后的MongoDB查询语句如下：


$qry= array(“id” => array(“$ne” => 2))
所以，结果就是MongoDB返回了除了id=2的其他所有数据。 

接下来我们看看另一种情况，通过脚本实现同样的功能。所不同的是，我们在后台用MongoDB中的findOne来查询结果。

我们先来快速看下MongoDB中的findOne方法：


db.collection.findOne(query, projection)
返回的是所有满足查询条件的文档中的第一个文档。如下图，但我们想要查询id=2的文档，输入以下语句：

图片29.png 

然后，我们看下php源代码：

图片30.png 

在这里，最关键的就是破坏原有的查询语句，再重新执行一个查询语句。

能想象到以下请求会在MongoDB中执行怎样的操作吗？

http://localhost/mongo/inject.php?u_name=dummy’});return{something:1,something:2}}//&u_pass=dummy
在这里，我们将原有的查询闭合了，然后返回了一个想要的参数，如下图：

图片31.png 

注意：报错的信息中向我们暴露了username和password这两个字段的存在，那么我们就把刚刚的注入语句改上username和password 参数，这正是我们想要的。

图片32.png 

现在如果我们想要知道数据库名。在MongoDB中，db.getName()方法可以查到数据库的名字，所以我们可以构造如下参数：

图片33.png
为了获取到数据库中的内容，我们首先要知道数据库中所有集合名。通过db.getCollectionNames()就能知道数据库中用的集合。

图片34.png 

好了，目前为止，我们获得了数据库名和集合名。现在需要做的就是获取到users集合中的数据，可以构造如下语句：

 图片35.png

我们可以用过改变数字来遍历整个集合，例如，改成 db.users.find()[2]，如下图：

图片36.png 

好了，现在也许大家已经了解这种注入方法。那么，该如何防御呢？

防御方法
我们回想下，第一个例子中的遍历是传递给一个数组的(array)。防御这种注入的话，我们总得先防止数组中的运算操作。因此，其中一种防御方法就是implode()方法，如下图：

 图片37.png

implode()函数返回由数组元素组合成的字符串。这样的话，我们就只能得到一个对应的结果。

图片38.png 

第二个例子我们可以使用addslashes()函数，这样的话攻击者就不能破坏查询语句了。同时，用正则表达式把一些特殊符号替换掉也是一个不错的选择。你可以使用如下正则：


$u_name =preg_replace(‘/[^a-z0-9]/i’, ‘\’, $_GET[‘u_name’]);
图片39.png 

现在再尝试下，就没有报错信息了。

图片40.png 

参考链接
http://php.net/manual/en/mongocollection.find.php

https://media.blackhat.com/bh-us-11/Sullivan/BH_US_11_Sullivan_Server_Side_WP.pdf
