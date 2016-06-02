---
layout: post
title: 用rabbitMQ做基于python的消息发送和接收
description: "怎样用rabbitMQ做消息的发送和接收，本文介绍它的一些简单原理，并介绍简单的实现代码"
modified: 2016-5-26
tags: [python, rabbitMQ]
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

怎么用python来写AMQP的信息收发，下面写几篇文章总结一下如何从发送接收消息到使用rabbitMQ做RPC。

ok，前面都是废话，本篇的后续内容，介绍如何实现消息在rabbitMQ上的发送接收。类似下面的这种最简单的情况。当然，准备在后面的两三篇文章讲到用rabbit来做python的rpc连接。
<figure class="half">
  <img src="/images/rabbit_trans.png" alt="">
  <figcaption>发送接收消息原理.</figcaption>
</figure>

我们用pika 0.10.0，这是一个对接rabbitMQ的python客户端包。


# 用python怎么写信息的发送端？
首先的首先，需要下载一个pika，地址https://pypi.python.org/pypi?:action=show_md5&digest=db5025bc5abfb0f78573616ee846df31
拿到之后，解压，安装，安装方法如下：
{% highlight bash %}
# python setup.py build
# python setup.py install
{% endhighlight %}
首先创建一个连接，假设MQ在本地。

{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
               'localhost'))
channel = connection.channel()
{% endhighlight %}

于是我们连接上了MQ。
（如果MQ有用户名密码，在哪里输入？这是个问题，下一篇文章解答）

连上之后，需要一个有一条队列，我们再往这个队列中扔信息。假设这个队列叫“hello”

{% highlight python %}
channel.queue_declare(queue='hello')
{% endhighlight %}

往队列里面发一条字符串“hello world!”

这个时候，通过命令rabbitmqctl list_queues可以看到你所发的这条消息数。如果你发送了消息另一端还没收走，应该有这种效果，hello队列里面有1条消息：
{% highlight python %}
$ rabbitmqctl list_queues
Listing queues ...
hello    1
...done.
{% endhighlight %}

# 用python怎么写信息的接收端？
同样，需要装好pika。
then, 需要在代码里创建一个callback函数，这个函数将被pika调用。
{% highlight python %}
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
{% endhighlight %}

然后，这个函数的接收对象队列需要说明清楚，是“hello”：
{% highlight python %}
channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
{% endhighlight %}

最后创建一个持续监听的进程，等待消息的到来。

## 消息确认机制
上面有个参数no_ack什么意思，解释一下。
消息被接收走之后，如果设置为no_ack，那此时rabbitMQ就会把队列里面这条消息干掉。如果此时任务执行失败，比如某个收了消息的worker挂了，那这个消息就从此消失了。因此，rabbitMQ考虑了这个情况，加了一个消息确认的机制，如果这里no_ack设置为False（或者不用设，默认ack开启），那么在消息被worker执行完返回的时候，会向队列发送一个ACK说明某个消息已经被接收和处理完了。MQ接收到这个ACK确认之后，才会将消息删除。一个场景：worker执行的时候宕机了，就会将消息发送给另一个worker，而不是直接删除。如此来保证消息被接收不丢失。
这里有个问题，rabbitMQ什么时候会认为worker挂掉而重发消息呢？判断标准是worker是否还在监听队列，如果它没有发送ACK就断开连接了，此时就认为消息该重发了。rabbitMQ没有超时的概念。
这个在接收端返回的ack如何发送呢？
{% highlight python %}
ch.basic_ack(delivery_tag = method.delivery_tag)
{% endhighlight %}
这句话加在callback函数的最后。如果忘了在接收端加这句重要的ACK而没设置no_ack=True的话，后果就会是，rabbitMQ一直以为你没有执行完成，于是一直hold住这个消息，这样消息体就会越塞越多，最后把内存撑爆。
这句话，用在下一篇文章里吧！

## 消息持久化
消息如果只存内存，那rabbitMQ主机倒了，上面的消息就全部丢失。消息的持久化可以避免这个问题。
首先，要在声明队列的时候加durable参数。这样可以使声明的‘hello’队列做持久化。
{% highlight python %}
channel.queue_declare(queue='hello', durable=True)
{% endhighlight %}
然后，在发送消息的时候，将消息的delivery_mode设为2，则该消息体也做了持久化。
{% highlight python %}
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      )) 
{% endhighlight %}

# 汇总
上面这两个特性先别用上，那这两段代码长这样：

## send.py
{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
{% endhighlight %}

## receive.py
{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
{% endhighlight %}

# 执行结果

## 发送端
{% highlight python %}
 # python sendamqp.py 
 [x] Sent 'Hello World!'
{% endhighlight %}

## 接收端
{% highlight python %}
  # python recamqp.py 
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Hello World!'
{% endhighlight %}

# 一些要点
1. 消息发送到exchange中，而不是直接发送到队列中。这个exchange暂时被默认设置为空，它其实可以用于发布和订阅的功能，下篇文章再介绍。
2. channel.queue_declare(queue='hello')这句将hello这个队列在发送程序和接收程序各自声明了一次，这是由于我们不能保证哪一个程序会被先执行。当然如果能保证其中一个先执行，另一个可以不用再声明。




