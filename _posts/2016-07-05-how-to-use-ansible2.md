---
layout: post
title: 如何使用ansible2 API
description: 
modified: 2016-6-29
tags: [ansible, python]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

在ansible1.9的时候，API是一个非常简单的东西。官方说“it's pretty simple”，真是又pretty又simple。
{% highlight python %}
import ansible.runner

runner = ansible.runner.Runner(
   module_name='ping',
   module_args='',
   pattern='web*',
   forks=10
)
datastructure = runner.run()
{% endhighlight %}

到了ansible2.0以后，是“a bit more complicated”，Oh my，简直让人难受。

简洁和灵活是鱼和熊掌。

# ansible2.0 API怎么用？

ansible2.0更贴近于ansible cli的常用命令执行方式，不同于上一版本只能发送单个命令或playbook；而更推荐用户在调用ansibleAPI的时候，将playbook的每个task拆分出来，获取每个task的结果。能够跟灵活处理在执行批量作业过程中的各种反馈。

* 将执行操作的队列模型，包含各类环境参数设置，归结到“ansible.executor.task_queue_manager”类中

* 将执行过程中的各个task的设置，或者说playbook中的编排内容，归结到“ansible.playbook.play”中

上述两个东西，几乎囊括了可以在执行过程中设置的所有参数，足够灵活，也让人抓狂，相当于需要自己写一个1.9版本中的runner。
他们的确也都是原生类，并非专用于外部调用。

# ansible.executor.task_queue_manager

task_queue_manager，是ansible的一个内部模块（ansible/executor/task_queue_manager.py）。初始化如下：
{% highlight python %}
class TaskQueueManager:

    '''
    This class handles the multiprocessing requirements of Ansible by
    creating a pool of worker forks, a result handler fork, and a
    manager object with shared datastructures/queues for coordinating
    work between all processes.

    The queue manager is responsible for loading the play strategy plugin,
    which dispatches the Play's tasks to hosts.
    '''

    def __init__(self, inventory, variable_manager, loader, options, passwords, stdout_callback=None, run_additional_callbacks=True, run_tree=False):

        self._inventory        = inventory
        self._variable_manager = variable_manager
        self._loader           = loader
        self._options          = options
        self._stats            = AggregateStats()
        self.passwords         = passwords
        self._stdout_callback  = stdout_callback
        self._run_additional_callbacks = run_additional_callbacks
        self._run_tree         = run_tree

        self._callbacks_loaded = False
        self._callback_plugins = []
        self._start_at_done    = False
        self._result_prc       = None

        ……
{% endhighlight %}

创建时，需要的主要参数包括：

* inventory  -->  由ansible.inventory模块创建，用于导入inventory文件

* variable_manager  -->  由ansible.vars模块创建，用于存储各类变量信息

* loader  -->  由ansible.parsing.dataloader模块创建，用于数据解析

* options  -->   存放各类配置信息的数据字典

* passwords  -->  登录密码，可设置加密信息

* stdout_callback  -->  回调函数

# ansible.playbook.play

ansible.playbook是一个原生模块，既用于CLI也用于API。这里可以看出来：

{% highlight python %}
try:
    from __main__ import display
except ImportError:
    from ansible.utils.display import Display
    display = Display()
{% endhighlight %}

ansible.playbook.play（ansible/playbook/play.py）。初始化如下：

{% highlight python %}
__all__ = ['Play']


class Play(Base, Taggable, Become):

    """
    A play is a language feature that represents a list of roles and/or
    task/handler blocks to execute on a given set of hosts.

    Usage:

       Play.load(datastructure) -> Play
       Play.something(...)
    """
{% endhighlight %}

* 最后，用task_queue_manager(play)来执行。

{% highlight python %}
def run(self, play):
        '''
        Iterates over the roles/tasks in a play, using the given (or default)
        strategy for queueing tasks. The default is the linear strategy, which
        operates like classic Ansible by keeping all hosts in lock-step with
        a given task (meaning no hosts move on to the next task until all hosts
        are done with the current task).
        '''
{% endhighlight %}

# 一个完整的流程

{% highlight python %}
# -*- coding:utf-8 -*-
# !/usr/bin/env python
#
# Author: Shawn.T
# Email: shawntai.ds@gmail.com
#
# this is the Interface package of Ansible2 API
#

from collections import namedtuple
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from tempfile import NamedTemporaryFile
import os

class AnsibleTask(object):
    def __init__(self, targetHost):
        Options = namedtuple(
                          'Options', [
                              'listtags', 'listtasks', 'listhosts', 'syntax', 'connection','module_path',
                              'forks', 'remote_user', 'private_key_file', 'ssh_common_args', 'ssh_extra_args',
                              'sftp_extra_args', 'scp_extra_args', 'become', 'become_method', 'become_user',
                              'verbosity', 'check'
                          ]
                       )

        # initialize needed objects
        self.variable_manager = VariableManager()

        self.options = Options(
                          listtags=False, listtasks=False, listhosts=False, syntax=False, connection='smart',
                          module_path='/usr/lib/python2.7/site-packages/ansible/modules', forks=100,
                          remote_user='root', private_key_file=None, ssh_common_args=None, ssh_extra_args=None,
                          sftp_extra_args=None, scp_extra_args=None, become=False, become_method=None, become_user='root',
                          verbosity=None, check=False
                      )
        self.passwords = dict(vault_pass='secret')
        self.loader = DataLoader()

		# create inventory and pass to var manager
        self.hostsFile = NamedTemporaryFile(delete=False)
        self.hostsFile.write(targetHost)
        self.hostsFile.close()
        self.inventory = Inventory(loader=self.loader, variable_manager=self.variable_manager, host_list=self.hostsFile.name)
        self.variable_manager.set_inventory(self.inventory)

    def ansiblePlay(self, action):
        # create play with tasks
        args = "ls /"
        play_source =  dict(
                name = "Ansible Play",
                hosts = 'all',
                gather_facts = 'no',
                tasks = [
                    dict(action=dict(module='shell', args=args), register='shell_out'),
                    dict(action=dict(module='debug', args=dict(msg='{{shell_out.stdout}}')))
                ]
            )
        play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)

    	# run it
        tqm = None
        try:
            tqm = TaskQueueManager(
                      inventory=self.inventory,
                      variable_manager=self.variable_manager,
                      loader=self.loader,
                      options=self.options,
                      passwords=self.passwords,
                      stdout_callback='default',
                  )
            result = tqm.run(play)
        finally:
        # print result
            if tqm is not None:
                tqm.cleanup()
                os.remove(self.hostsFile.name)
                self.inventory.clear_pattern_cache()
            return result

{% endhighlight %}

写一个ansibleTask类，创建了上述的各类必要的配置信息对象，最后使用ansibleTask.ansiblePlay()函数执行。

* inventory文件的动态生成

写上面的代码的过程中，碰到一个问题：inventory对象创建时需要一个实体的hosts文件，而文件需要动态生成。
生成的方法参考了<a href="https://serversforhackers.com/running-ansible-2-programmatically">这篇牛逼闪闪的文章</a>。使用tempfile.NamedTemporaryFile这个方法来创建一个有名称的临时文件，可以选择关闭后删除或保留。
