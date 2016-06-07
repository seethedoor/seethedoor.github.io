---
layout: post
title: openstack的nova image-list错误
description: "解决一个nova image-list的错误"
modified: 2016-6-2
tags: [openstack, nova, glance]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

# 问题描述
* 安装openstack kilo版的时候有一个十分诡异的现象，在使用glance client执行glance image-list时十分正常，而在使用nova client执行nova image-list时却返回一个500：

{%highlight shell%}
# nova image-list                                
ERROR (ClientException): The server has either erred or is incapable of performing the requested operation. (HTTP 500) (Request-ID: req-2cd7b9bd-6d8c-461f-b25b-83fcac8ae3d4)
{%endhighlight%}

# 解决
找到这个文章（https://ask.openstack.org/en/question/66759/nova-image-list-returns-http-500/），这个问题似乎只会发生在使用了诸如haporxy等负载均衡的组件上，这个组件为glance-registry注册了一个vip。我们在glance的配置文件glance-api.conf的
{%highlight shell%}
[DEFAULT]
registry_host=xxx
{%endhighlight%}
中，需要使用此vip地址。否则出此500错误。