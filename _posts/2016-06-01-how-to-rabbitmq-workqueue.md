---
layout: post
title: 用rabbitMQ做工作队列的接收端负载均衡
description: "怎样用rabbitMQ来做python的工作队列，用于解决一些时延性的方法调度问题"
modified: 2016-6-1
tags: [python, rabbitMQ]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

有一些消息并不是马上能够回应的，往往我们会考虑采用在处理端多起几个服务的方法，让工作分担到多个服务上去执行。这个任务可以用rabbitMQ的workqueue来解决。
把上一篇文章中的sender.py和receiver.py做一些调整，给队列发送一些自定义的字符串。

## sender.py
{% highlight css %}
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print " [x] Sent %r" % (message,) 
{% endhighlight %}

对接收端也做一些修改，将它改为比较耗时的任务。这个改动会计算消息体中含有的‘.’的个数，来模拟任务处理的持续时间秒数。
## worker.py
{% highlight css %}
import time

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)
{% endhighlight %}

## 效果试验
启动四个ssh终端，一个用于多次执行task.py发送消息，另外三个执行worker.py的守护进程。尝试发送6个任务，每个worker各自执行了两个任务。
worker1
{% highlight css %}
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received '1.........'
 [x] Done
 [x] Received '4......'
 [x] Done
{% endhighlight %}

worker2
{% highlight css %}
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received '2.........'
 [x] Done
 [x] Received '5......'
 [x] Done
{% endhighlight %}

worker3
{% highlight css %}
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received '3.........'
 [x] Done
 [x] Received '6......'
 [x] Done
{% endhighlight %}

上面的结果，显然是顺序接收，使用了轮询的消息处理（round-robin）。这种方式有一个问题，只是一味地进行平均分发，如果任务的长短并不相同，那负载均衡效果就比较差。
rabbitMQ有一个basic_qos方法，将prefecth_count设置为1，表示在worker处理完手头的1个任务之前，不给他发送其他消息。这样，可以让MQ优先发给那些先处理完手头任务的接收者。
{% highlight css %}
channel.basic_qos(prefecth_count=1)
{% endhighlight %}

不过这种办法要考虑队列本身能承受的消息量。如果消息量比较大，还是尽快将消息分发下去为好，免得把rabbitMQ撑爆，性能下降。

# 代码汇总

## task.py
{% highlight css %}
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

## worker.py
{% highlight css %}
#!/usr/bin/env python
import pika,time

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

print(' [*] Waiting for messages. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='task_queue')

channel.start_consuming()
{% endhighlight %}