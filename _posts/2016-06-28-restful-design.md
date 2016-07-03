---
layout: post
title: 用restful来设计API
description: 
modified: 2016-6-28
tags: [others]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

大家都说要用Restful的API，Restful到底长啥样？

# Restful的原则和方法

Restful认为互联网的世界一切皆为资源。API的操作都是对资源的增删改查。

* POST 新增一个完整的资源，多次执行可能会产生重复资源
* PUT 更新一个完整的资源，重复执行不会产生重复的资源
* PATCH 更新资源中的部分内容
* GET 获取资源，对资源无影响
* DELETE 删除整个资源

事实上，采用Restful这事儿是针对Soap来讲的。人家用的接口灵活程度高，可以是各种自由的方法。但是自由往往带来的另一个负面的影响就是复杂度高，这是一个哲学命题，有空可以慢慢探讨。不过在我们谈的这个话题下是完全成立的。

理由很简单，如果可以有各种自由的方法，那定义方法的时候将需要更多的附属信息；方法的理解难度也随之提高，API需要撰写一大坨的文档。

如果能把所有的方法统一为上面的五个维度，带来的简洁、便利性可想而知。但前提是你的每个方法都能这么做。

所以这是它带来的问题，碰到无法改为restful的方法，怎么改都会觉得自己的方法变得十分牵强的时候，还是十分困扰的。
这是我对Restful的思想理解不够深入？
事实上Restful不能包治百病，实际用起来还是需要适当灵活调整。