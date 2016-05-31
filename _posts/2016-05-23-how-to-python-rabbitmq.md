---
layout: post
title: 用rabbitMQ做python的amqp连接
description: "怎样用rabbitMQ来做python的amqp协议的分布式rpc连接，本文介绍它的一些简单原理，并介绍简单的实现代码"
modified: 2016-5-26
tags: [python]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

注意：这篇博客很多内容翻译自rabbitMQ的官网:-)。

现在的后台应用，稍有一点点规模，就该扯上分布式了。这时候会涉及一个最基本的问题，分布在不同主机上，甚至可能异构的环境下，怎样让远端的服务、方法，甚至对象得到调用，更甚者，不影响总体框架，对上层透明。

有很多方法。

最常用的叫RPC，远程过程调用。按照传输的数据格式，又分为xml-rpc和json-rpc。当然各有利弊，实现的方法在上一篇博客做了介绍。然而它有一点需要提及的，是RPC需要在服务端建立http服务，需要一个web容器和程序解析器。还有类似的，RMI，远程对象调用，本文不打算细谈。

第二个叫AMQP。AMQP是一个协议，常用的是RabbitMQ、zeroMQ等。严格来说，是一个比RPC更为复杂的RPC，它将所有的请求收集起来，在分发出去。类似于一个post office。

Post Office当然是一个第三方的服务。于是不管是服务端还是客户端，是收信件的还是发信件的，大家都要跟这个Post Office连接上。post office会根据发信人的地址，把信送到收信人的手里。这个过程，包括了信息收发、信息存储、信息路由。靠谱的post office会把所有信件存在保险箱里，就算房子倒了，保险箱里的信件还在。rabbitMQ就是这种，它会把信息做持久化，不是只存在内存中，重启主机还能找回数据。有的MQ没有持久化，比如ActiveMQ。

下面是代码，怎么用python来写AMQP的信息收发。我们用pika 0.10.0，这是一个AMQP协议的python客户端包。

# 用python怎么写信息的发送端？

首先创建一个连接，假设MQ在本地。

{% highlight css %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
               'localhost'))
channel = connection.channel()
{% endhighlight %}

于是我们连接上了MQ。
我要提问：MQ有用户名密码，在哪里输入？

连上之后，需要一个有一条队列，我们再往这个队列中扔信息。假设这个队列叫“hello”

{% highlight css %}
channel.queue_declare(queue='hello')
{% endhighlight %}

往队列里面发一条字符串“hello world!”




