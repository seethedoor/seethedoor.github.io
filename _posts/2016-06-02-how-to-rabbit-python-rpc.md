---
layout: post
title: 使用rabbitMQ实现python服务的rpc连接
description: "使用rabbitMQ实现python服务的rpc连接"
modified: 2016-6-2
tags: [python, rabbitMQ, rpc]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

# 原理
* 当客户端启动的时候，它创建一个匿名独享的回调队列。
* 在RPC请求中，客户端发送带有两个属性的消息：一个是设置回调队列的reply_to属性，另一个是设置唯一值的correlation_id属性。
其中，reply_to属性代表了回调队列名称，correlatione_id用于发送请求时标志本次请求的id号。
将请求发送到一个 rpc_queue 队列中。
* RPC服务器端等待客户端将请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给reply_to字段指定的队列。如果队列设置为no_ack=False，则此时客户端会想MQ发送接收并完成任务的ACK，MQ接收到此ACK后将此消息删除（如果MQ在接收到ACK之前PRC服务端断连，则此消息会重发到其他服务端，不论服务端是否已经执行或是否已经将结果返回到reply_to队列；因此客户端需要合理处理重复的响应，服务端要合理处理重复的请求）。
* 客户端等待回调队列里的数据。当有消息出现的时候，它会检查correlation_id属性。如果此属性的值与请求时发送的值匹配，认为这个返回的消息确实是我请求的，此时客户端可以将结果反馈给上层应用。

<figure>
  <img src="/images/rabbit_rpc.png" alt="">
  <figcaption>PRC调用及回调原理.</figcaption>
</figure>

# 代码示例

## rpc_server.py
{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))

channel = connection.channel()
# 声明一个队列，用于接收客户端的rpc请求，服务端从此队列获取rpc请求
channel.queue_declare(queue='rpc_queue')
# 定义一个fibonachi函数作为服务端的计算内容
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-1) + fib(n-2)
# 定义一个函数，接收到客户端的请求消息后的响应
def on_request(ch, method, props, body):
    n = int(body)
    # 计算出客户端所需的结果
    print " [.] fib(%s)"  % (n,)
    response = fib(n)
    # 向reply_to队列返回执行结果，使用空名称topic交换机、props.reply_to的topic，附带的correlation_id信息，是从客户端的请求信息中带过来的
    ch.basic_publish(exchange='',
                     routing_key=props.reply_to,
                     properties=pika.BasicProperties(correlation_id = \
                                                     props.correlation_id),
                     body=str(response))
    # 向队列回复ACK，告知队列可以释放这条消息了
    ch.basic_ack(delivery_tag = method.delivery_tag)
# 设置服务端的负载均衡模式为，有一个消息在处理时，阻塞消息接收，请MQ将请求发送给其他服务进程
channel.basic_qos(prefetch_count=1)

# 从‘rpc_queue’队列取得消息，执行消息消费过程
channel.basic_consume(on_request, queue='rpc_queue')

print " [x] Awaiting RPC requests"
channel.start_consuming()

{% endhighlight %}

## rpc_client.py
{% highlight python %}
#!/usr/bin/env python
import pika
import uuid
# 调用PRC服务的客户端类
class FibonacciRpcClient(object):
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters(
                host='localhost'))
        
        self.channel = self.connection.channel()
        # 声明一个随机名称队列，队列为独享队列。此队列作为回调所用的reply_to队列，供rpc服务端发送响应信息。
        result = self.channel.queue_declare(exclusive=True)
        self.callback_queue = result.method.queue
        # 初始化接收响应信息的队列设置，包括无需返回响应、设定接收对象队列
        self.channel.basic_consume(self.on_response, no_ack=True,
                                   queue=self.callback_queue)

    # 验证响应信息中的correlation_id是否合法
    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    # 调用PRC服务的主函数
    def call(self, n):
        self.response = None
        # 创建rpc请求的唯一的correlation_id
        self.corr_id = str(uuid.uuid4())
        # 执行消息发送，设定交换机为空名称交换机、设定routing_key、设定回调信息队列、设定响应时信息需要携带的correlation_id
        self.channel.basic_publish(exchange='',
                                   routing_key='rpc_queue',
                                   properties=pika.BasicProperties(
                                         reply_to = self.callback_queue,
                                         correlation_id = self.corr_id,
                                         ),
                                   body=str(n))
        while self.response is None:
            self.connection.process_data_events()
        return int(self.response)
# 执行PRC调用
fibonacci_rpc = FibonacciRpcClient()

print " [x] Requesting fib(30)"
response = fibonacci_rpc.call(30)
print " [.] Got %r" % (response,)
{% endhighlight %}
