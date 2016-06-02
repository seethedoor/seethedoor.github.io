---
layout: post
title: 使用rabbitMQ的消息交换机
description: "rabbitMQ的消息交换机制"
modified: 2016-6-2
tags: [python, rabbitMQ]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

消息并不是直接从消息生产者发到队列中，而是发送到exchange--一个消息交换机，再从消息交换机发送给特定的队列。这里的交换机有几种交换机制。

## fanout exchange
扇形交换机。相当于广播，把exchange收到的消息广播到绑定在这个交换机上的队列。

## direct exchange
* 直连交换机。它会将消息的路由键（消息的routing_key，发送消息时设置）和队列的绑定键(队列的routing_key，声明队列时设置)比较，匹配上了就发到该队列上。
* 多个队列可以使用相同的绑定键，效果就类似一个局部的扇形交换机。

## topic exchange
主题交换机。扇形交换机太过粗放，直连交换机太过死板，这时候需要主题交换机出马。它可以给路由键创建多个角度的信息，而绑定键的匹配对象可以只涉及其中的一部分也可以是全部。例如：
* 路由键为 male.old.tall。他能够匹配到的绑定键如：male.#、*.old.*、#.tall等，而如male.*.*.*则匹配不了。
* 这里用到的符号，*代表一个单词，#代表任意多个单词；只有一个#代表匹配所有路由键。

## headers exchange




# 扇形交换机示例

<figure class="half">
  <img src="/images/rabbit_fanout.png" alt="">
  <figcaption>扇形交换机原理.</figcaption>
</figure>

## emi_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明一个交换机，这个交换机的名称是logs,类型是扇形交换机
channel.exchange_declare(exchange='logs',
                         type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"

# 将message发送到logs交换机，routing_key为默认空值，因为是扇形交换机，不会进行routing_key与队列的binding_key匹配。
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
print " [x] Sent %r" % (message,)
connection.close()
{% endhighlight %}

## receiv_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明交换机
channel.exchange_declare(exchange='logs',
                         type='fanout')

# 队列声明，不指定队列名，队列名将由系统随机生成；将队列设为独有，则消费者断开时，队列将被删除。
result = channel.queue_declare(exclusive=True)

# 取得队列名称，因上句中队列名为系统随机生成，队列名通过下面这句获得。
queue_name = result.method.queue

# 将队列绑定到‘logs’交换机
channel.queue_bind(exchange='logs',
                   queue=queue_name)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r" % (body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
{% endhighlight %}

# 直连交换机示例
<figure class="half">
  <img src="/images/rabbit_direct.png" alt="">
  <figcaption>直连交换机原理.</figcaption>
</figure>

## emi_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明一个名为‘direct_logs’的交换机，类型为直连交换机
channel.exchange_declare(exchange='direct_logs',
                         type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

# 发布消息到‘direct_logs’交换机，路由键是serverity
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)

print " [x] Sent %r:%r" % (severity, message)
connection.close()
{% endhighlight %}

## receiv_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明交换机
channel.exchange_declare(exchange='direct_logs',
                         type='direct')

# 声明随机名称队列，并设为独享
result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

# 获取本程序的消息类型
severities = sys.argv[1:]
if not severities:
    print >> sys.stderr, "Usage: %s [info] [warning] [error]" % \
                         (sys.argv[0],)
    sys.exit(1)

for severity in severities:
    # 对交换机做绑定
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r:%r" % (method.routing_key, body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
{% endhighlight %}

# 主题交换机示例
<figure class="half">
  <img src="/images/rabbit_topic.png" alt="">
  <figcaption>主题交换机原理.</figcaption>
</figure>

## emi_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明一个主题交换机，名称是topic
channel.exchange_declare(exchange='topic_logs',
                         type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 1 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

# 消息发送，设置了routing_key
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print " [x] Sent %r:%r" % (routing_key, message)
connection.close()
{% endhighlight %}

## receiv_logs.py
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

# 声明交换机和队列
channel.exchange_declare(exchange='topic_logs',
                         type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    print >> sys.stderr, "Usage: %s [binding_key]..." % (sys.argv[0],)
    sys.exit(1)

for binding_key in binding_keys:
# 设置队列的绑定
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r:%r" % (method.routing_key, body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

{% endhighlight %}
