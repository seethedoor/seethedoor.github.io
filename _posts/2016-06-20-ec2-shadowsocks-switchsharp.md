---
layout: post
title: 在AWS-EC2搭建shadowsocks前向代理
description: 在AWS-EC2搭建shadowsocks前向代理，顺便用switchsharp做访问规则切换
modified: 2016-6-20
tags: [forward proxy]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

这个十分实用，废话不多说了：就是教你在国外亚马逊搭一套shadowsocks，然后本地用chrome和插件“switchsharp”科学上网。
（以下内容从几篇文章里面转载整合）

# 首先，购买aws云主机
1. 在aws.amazon.com注册账号，注册过程将需要信用卡预付费$1，并且完成电话认证。
2. 进入aws控制台，创建一个keypair，将下载的keypair文件放在本地的~/.ssh中。
3. 进入ec2，创建一台免费的centos7.1云主机；并增加一条安全策略：inband和outband都为
ALLTCP  TCP 0-65535 Anywhere 0.0.0.0/0
4. 配置本地ssh登录：
vim ~/.ssh/config
增加：
{% highlight shell %}
Host aws_ec2
        HostName <云主机公网IPXX.XX.XX.XX>
        User ec2-user
        IdentityFile ~/.ssh/osapp.pem
{% endhighlight %}
至此可以在本地主机ssh aws_ec2登录远程主机搭建shadowsocks

# 然后在远程主机安装shadowsocks服务

1、安装组件，运行以下指令：
{% highlight shell %}
yum install m2crypto python-setuptools
easy_install pip
pip install shadowsocks
{% endhighlight %}
安装时部分组件需要输入Y确认。

2、安装完成后配置服务器参数，运行以下指令：
{% highlight shell %}
vi  /etc/shadowsocks.json
{% endhighlight %}

写入配置如下：
{% highlight shell %}
{
    "server":"0.0.0.0",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}
{% endhighlight %}

将上面的mypassword替换成你的密码，server_port也是可以修改的，例如3024，传说端口越小，效果越好，这个我没有去验证，但建议不要小于1024，以免引起不必要的麻烦。

3、运行下面的命令，启动shadowsocks

{% highlight shell %}
ssserver -c /etc/shadowsocks.json
{% endhighlight %}

# 本地配置
1. 安装shadowsocks客户端，在“服务器-服务器设定”里添加一个服务器，地址写ec2服务器的公网地址，端口、密码按照上面的设置写。加密方式为上述设置中的method。设置完了连接上，便可使用。
2. 为了方便切换网址访问采用的方式（例如一般国内网站使用直接访问，国外网站通过shadowsocks），在本地安装chrome并安装插件switchsharp。switchsharp可以方便地设置各种访问规则，非常好用。配置方法：在插件中设置“手动配置”socket：127.0.0.1：1080  （选择socket5）

至此可以科学上网。

# 最后，安装进程控制工具supervisor
将shadowsocks加入supervisor进程控制，否则其受限于本地ssh进程。

1、安装supervisor，运行以下命令

{% highlight shell %}
yum install python-setuptools
easy_install supervisor
{% endhighlight %}

然后创建配置文件

{% highlight shell %}
echo_supervisord_conf > /etc/supervisord.conf
{% endhighlight %}

修改配置文件
{% highlight shell %}
vi /etc/supervisord.conf
{% endhighlight %}

在文件末尾添加
{% highlight shell %}
[program:ssserver]
command = ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
startsecs=3
{% endhighlight %}

配置文件说明
　　运行命令：
{% highlight shell %}
supervisord
{% endhighlight %}

2、设置supervisord开机启动
　　编辑文件：
{% highlight shell %}
vi /etc/rc.local
{% endhighlight %}

在末尾另起一行添加
{% highlight shell %}
supervisord
{% endhighlight %}

保存退出（和上文类似）。
另centos7还需要为rc.local添加执行权限

{% highlight shell %}
chmod +x /etc/rc.local
{% endhighlight %}

至此运用supervisord控制shadowsocks开机自启和后台运行设置完成
