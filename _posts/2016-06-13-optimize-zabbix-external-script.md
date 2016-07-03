---
layout: post
title: 使用zabbix监控openstack RPC服务的一种方法
description: "也可作为一个zabbix外部监控脚本优化的实例"
modified: 2016-6-1
tags: [python, zabbix, MQ]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

处理完了python和rabbitMQ的连接问题，除了自己开发的需要，也是为了找到对openstack中各个rpc服务进行监控的办法。

由于openstack的每个RPC服务都在rabbitMQ中注册了一个自己的队列。而且有队列的汇总名称，多个相同的服务又会有子队列。
于是我考虑了这样一种方法，利用rabbitMQ自带的管理工具rabbitmqctl，把主队列包含的consumers个数罗列出来，如果这些consumers个数在合理范围内（例如conductor队列有5个consumer，而我们刚好部署了5套nova-conductor在5个主机上），我认为目前这个RPC服务是正常的。如果这个主队列中的consumer个数过少（例如conductor队列只有1个consumer了，甚至查不到consumer，所有consumer都断开了对rabbitMQ的连接），那必须认为目前这些RPC服务已经出了问题。

这个找到所有队列中consumer个数的命令是：
{% highlight shell %}
# ./rabbitmqctl list_queues name consumers
{% endhighlight %}

将获取到的列表进行搜索，匹配那些你关心的RPC服务对应的queue（这个queue name往往就是RPC服务的名称），将它后面的数字取回来，它就是consumer的个数。这个数字可以返回作为zabbix监控项的指标值，将其进行阈值匹配产生告警。

* 问题1
这里碰到zabbix的典型问题：脚本执行时间太长。

脚本执行时间太长，将导致zabbix server的poller进程负荷过大，因其需保持此连接，等待脚本的执行完成。解决的办法，通过crontab将执行结果保存在指定文件，让监控脚本去读文件获取结果。
{% highlight shell %}
# crontab -e
*/1 * * * * /apps/svr/rabbitmq_5672/sbin/rabbitmqctl list_queues name consumers > /tmp/queues
{% endhighlight %}

* 问题2
通过上述方法，又产生了第二个问题：
在crontab中定时每分钟执行的命令，它完全打印完结果需要一定的时间（因为queues往往比较多），如果在打印未完成时zabbix要执行监控脚本，则脚本将返回错误结果。
处理方法：
将queue列表保存在一个临时文件中，保存完了再存到监控脚本将要读取的文件里。这只需要更新一下crontab任务：
{% highlight shell %}
# crontab -e
*/1 * * * * /apps/svr/rabbitmq_5672/sbin/rabbitmqctl list_queues name consumers > /tmp/tmp-queues;mv /tmp/tmp-queues /tmp/queues
{% endhighlight %}

* 监控脚本
脚本的核心几行代码无非就是做了一个python的正则检索。
{% highlight python %}
        #(status,result) = commands.getstatusoutput('/apps/svr/rabbitmq_5672/sbin/rabbitmqctl list_queues name consumers|grep senlin')
        (status,result) = commands.getstatusoutput('cat /tmp/queues')

        # when command ran faild
        if status is not 0:
            njInfo['errLog'] = 'cmd faild'
            return njInfo
        # when command ran successfully
        njInfo['status'] = True
        njInfo['content']['consumerNum'] = {}
        # target queue name
        rpcList = {
            'alarm_notifier',
            'notifications.error',
            'notifications.info',
            'cert',
            'scheduler',
            'conductor',
            'console',
            'consoleauth',
            'compute',
            'engine',
            'heat-engine-listener',
            'cinder-backup',
            'cinder-scheduler',
            'cinder-volume',
            'metering',
            'senlin-engine'
            }
        for rpcItem in rpcList:
            searchObj = re.compile(rpcItem+'\\t([0-9]+)', re.S)
            matchObj = searchObj.search(result)
            if not matchObj:
                njInfo['errLog'] = njInfo['errLog'] + 'result donnot match "' + rpcItem +'"\n'
            else:
                njInfo['content']['consumerNum'][rpcItem] = matchObj.group(1)
        # search results is in matchObj.group, take it out.
{% endhighlight %}

当然这段代码返回的是一堆queue的comsumer数，zabbix的监控往往只要一个，那么需要再做一层匹配啦！
