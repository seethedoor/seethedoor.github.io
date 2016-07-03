---
layout: post
title: 使用python/flask实现认证、鉴权
description: 
modified: 2016-6-29
tags: [auth, python]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

上一篇文章介绍了使用JWT协议来做token认证的功能，继续装逼下去，怎样实现一个认证&鉴权的模块？

假设token认证的功能已经完成，那么设定为每次请求头部中都带上这个token。server端在每次响应请求时都要做一次token的校验，包括两个内容：

* token是否合法，token是否过期、是否被篡改，token内的信息是否有效
* 用户是否有权限执行此请求

在python/flash上，这个检验使用decorator实现，会是一个很靠谱的办法，代码简洁性和可读性都很不错。

# 为认证和鉴权创建数据模型
首先，理清关系，用户-角色-权限，是多对多对多的关系。通过flask-sqlalchemy给这三者创建多对多模型，会是五张表，其中有两张表是多对多连接用的实体表，无ORM模型（根据flask的建议）。

然后，为了实现审计功能（一些删除对象操作之后将造成历史数据无法对应），用户-角色-权限都需要添加valid字段。删除时只设为不可用。

最后，创建一个decorator函数，实现检查token并审核权限的功能。

# 目标效果是怎样的？

在一个api或函数前，添加此decorator，参数包含它所需要的权限名称。一旦请求执行到此函数，将执行decorator的权限检查。如果认证或鉴权失败，直接走异常处理流程，不会执行请求内容。类似这样：

{% highlight python %}
@auth.PrivilegeAuth(privilegeRequired="userAdmin")
    def get(self):
        """
        get user list or one user info ...
        """
{% endhighlight %}

# 怎么实现

这需要一个带参数的decorator类，需要传参。内部逻辑十分简单，检查token--检查用户--检查角色--检查权限匹配--搞定放行。

ps.抱歉我的英文十分chinglish哈哈。。

auth.py
{% highlight python %}
class PrivilegeAuth(Resource):
    """
    This class is used tobe a decoretor for other methods to check the
    client's privilege by token.
    Your method's 'required privilege' should be set as an argument of this
    decoretor. And this 'required privilege' should have been in the
    'privilege' table.
    If your method's 'required privilege' is one of user's privileges,
    user will be allowed to access the method, otherwise not.
    ps. user's privilege is checked by his token.
    """
    def __init__(self, privilegeRequired):
        # argument 'privilegeRequired' is to set up your method's privilege
        # name
        self.privilegeRequired = privilegeRequired
        super(PrivilegeAuth, self).__init__()

    def __call__(self, fn):
        def wrapped(*args, **kwargs):
            try:
                rolesReq = Privilege.getFromPrivilegeName(
                    self.privilegeRequired).roles
            except:
                msg = 'wrong privilege setting: privilege (' +\
                    self.privilegeRequired + ') doesnot set in any roles'
                app.logger.error(utils.logmsg(msg))
                raise utils.InvalidModuleUsage('wrong privilege setting')

            # get user's privileges by his token
            # if token is in body, then:
            # myreqparse = reqparse.RequestParser()
            # myreqparse.add_argument('token')
            # args = myreqparse.parse_args()
            # if token is in headers, then:
            token = request.headers.get('token')
            if not token:
                msg = "you need a token to access"
                raise utils.InvalidAPIUsage(msg)
            [userId, roleIdList, msg] = User.tokenAuth(token)
            if not userId:
                msg = msg + " when autherization"
                raise utils.InvalidAPIUsage(msg)
            else:
                currentUser = User.getValidUser(userId=userId)
                if not currentUser:
                    msg = "cannot find user when autherization"
                    raise utils.InvalidAPIUsage(msg)

            # user's privilege auth
            for role in currentUser.roles:
                for roleReq in rolesReq:
                    if role.role_id == roleReq.role_id:
                        return fn(*args, **kwargs)

            msg = "Privilege not Allowed."
            app.logger.info(utils.logmsg(msg))
            raise utils.InvalidAPIUsage(msg)
        return wrapped

{% endhighlight %}
