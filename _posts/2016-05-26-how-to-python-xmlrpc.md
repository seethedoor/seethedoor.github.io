---
layout: post
title: 如何使用python创建xml-RPC服务
description: "怎样使用python的自有模块完成xml-rpc服务端和客户端，并提供了实例代码"
modified: 2016-5-24
tags: [python]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---


关于rpc还是restful的问题争议了很多年，这个文章没有打算谈这个问题，存在即合理吧。这里只打算解决一下怎么用python自有的模块实现xml-rpc。

# what is RPC?

Remote Procedure Call， 远程过程调用， 一种常用的分布式系统接口对接方式， 简称RPC。 可以把远程的方法开放到网络上， 通过http、 amqp等形式做连接， 数据结构有xml/json等。 xml作为传输格式的优势在于格式灵活，但内容较重，且不适合于与前端对接。

采用SimpleXMLRPCServer模块可实现python的xml-RPC服务。测试时仍然在同一台主机，通过127.0.0.1访问。


# how to use python xml-RPC?

这里做了个最简单的例子，实现了加减乘除四个方法。server端脚本如下:

{% highlight css %}
# -*- coding:utf-8 -*-
# !/usr/bin/env python
from SimpleXMLRPCServer import SimpleXMLRPCServer
def add(x,y):
    return x+y
 
def subtract(x, y):
    return x-y
 
def multiply(x, y):
    return x*y
 
def divide(x, y):
    return x/y
 
 
server = SimpleXMLRPCServer(("127.0.0.1", 8000))
print "Listening on port 8000..."
server.register_multicall_functions()
server.register_function(add, 'add')
server.register_function(subtract, 'subtract')
server.register_function(multiply, 'multiply')
server.register_function(divide, 'divide')
server.serve_forever()
{% endhighlight %}

client端则远程调用这四个函数并且打印结果，脚本如下：

{% highlight css %}
# -*- coding:utf-8 -*-
# !/usr/bin/env python

import xmlrpclib
 
proxy = xmlrpclib.ServerProxy("http://localhost:8000/")
multicall = xmlrpclib.MultiCall(proxy)
multicall.add(7,3)
multicall.subtract(7,3)
multicall.multiply(7,3)
multicall.divide(7,3)
result = multicall()

print "7+3=%d, 7-3=%d, 7*3=%d, 7/3=%d" % tuple(result)
{% endhighlight %}


