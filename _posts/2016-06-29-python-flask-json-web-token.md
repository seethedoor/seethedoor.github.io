---
layout: post
title: 使用python实现后台系统的JWT认证
description: 
modified: 2016-6-29
tags: [JWT python flask]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

前段时间给一个运维系统的后端搭建代码框架，今天的文章介绍一种适用于restful+json的API认证方法，这个方法是基于jwt，并且加入了一些从oauth2.0借鉴的改良。

## 1. 常见的几种实现认证的方法

首先要明白，认证和鉴权是不同的。认证是判定用户的合法性，鉴权是判定用户的权限级别是否可执行后续操作。这里所讲的仅含认证。认证有几种方法：

### 1.1 basic auth

这是http协议中所带带基本认证，是一种简单为上的认证方式。原理是在每个请求的header中添加用户名和密码的字符串（格式为“username:password”，用base64编码）。

这种方式相当于将“用户名：密码”绑定为一个开放式证书，这会有几个问题：

* 每次请求都需要用户名密码，如果此连接未使用SSL/TLS，或加密被破解，用户名密码基本就暴露了；

* 无法注销用户的登录状态；

* 证书不会过期，除非修改密码。

总体来说，这种方法的特点就是，简单但不安全。

### 1.2cookie

将认证的结果存在客户端的cookie中，通过检查cookie中的身份信息来作为认证结果。
这种方式的特点是便捷，且只需要一次认证，多次可用；也可以注销登录状态和设置过期时间；甚至也有办法（比如设置httpOnly）来避免XSS攻击。

但它的缺点十分明显，使用cookie那便是有状态的服务了。

### 1.3 token

JWT协议似乎已经应用十分广泛，JSON Web Token——一种基于token的json格式web认证方法。基本的原理是，第一次认证通过用户名密码，服务端签发一个json格式的token。后续客户端的请求都携带这个token，服务端仅需要解析这个token，来判别客户端的身份和合法性。

而JWT协议仅仅规定了这个协议的格式（<a href="https://tools.ietf.org/heml/rfc7519">RFC7519</a>），它的序列生成方法在JWS协议中描述（https://tools.ietf.org/html/rfc7515），分为三个部分：

#### 1.3.1 header头部：

* 声明类型，这里是jwt

* 声明加密的算法 通常直接使用 HMAC SHA256

一种常见的头部是这样的：
{% highlight shell %}
{
  'typ': 'JWT',
  'alg': 'HS256'
}
{% endhighlight %}
再将其进行base64编码。

#### 1.3.2 payload载荷：
payload是放置实际有效使用信息的地方。JWT定义了几种内容，包括：

* 标准中注册的声明，如签发者，接收者，有效时间（exp），时间戳（iat,issued at）等；为官方建议但非必须
* 公共声明
* 私有声明

一个常见的payload是这样的：
{% highlight shell %}
{'user_id': 123456,
 'user_role': admin,
 'iat': 1467255177}
{% endhighlight %}
事实上，payload中的内容是自由的，按照自己开发的需要加入。

Ps.有个小问题。使用itsdangerous包的TimedJSONWebSignatureSerializer进行token序列生成的结果，exp是在头部里的。这里似乎违背了jwt的协议规则。

#### 1.3.3 signature
存储了序列化的secreate key和salt key。这个部分需要base64加密后的header和base64加密后的payload使用.连接组成的字符串，然后通过header中声明的加密方式进行加盐secret组合加密，然后就构成了jwt的第三部分。

## 2. 我的认证需求

考虑到我的系统是一个前后端分离的后台程序，用于运维工作，虽在内网使用，也有一定的保密性要求。

* API为restful+json的无状态接口，要求认证也是相同模式
* 可横向扩展
* 较低数据库压力
* 证书可注销
* 证书可自动延期

选择JWT。

## 3. JWT实现

### 2.1 如何生成token
这里使用python模块itsdangerous，这个模块能做很多编码工作，其中一个是实现JWS的token序列。
genTokenSeq这个函数用于生成token。其中使用的是TimedJSONWebSignatureSerializer进行序列的生成，这里secret_key、salt从配置文件中读取，当然也可以直接写死在这里。expires_in是超时时间间隔，这个间隔以秒记，可以直接在这里设置，我选择将其设为方法的形参（因为这个函数也用在了解决问题2）。

{% highlight python %}

# serializer for JWT
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer


    """
    token is generated as the JWT protocol.
    JSON Web Tokens(JWT) are an open, industry standard RFC 7519 method
    """
    def genTokenSeq(self, expires):
        s = Serializer(
            secret_key=app.config['SECRET_KEY'],
            salt=app.config['AUTH_SALT'],
            expires_in=expires)
        timestamp = time.time()
        return s.dumps(
            {'user_id': self.user_id,
             'user_role': self.role_id,
             'iat': timestamp})
        # The token contains userid, user role and the token generation time.
        # u can add sth more inside, if needed.
        # 'iat' means 'issued at'. claimed in JWT.
{% endhighlight %}

使用这个Serializer可以帮我们处理好header、signature的问题。我们只需要用s.dumps将payload的内容写进来。这里我准备在每个token中写入三个值：用户id、用户角色id和当前时间（‘iat’是JWT标准注册声明中的一项）。

假设我所写入的信息是
{% highlight shell %}
{
  "iat": 1467271277.131803,
  "user_id": "46501228343b11e6aaa6a45e60ed5ed5f973ba0fcf783bb8ade34c7b492d9e55",
  "user_role": 3
}
{% endhighlight %}
采用以上的方法所生成的token为
{% highlight shell %}
eyJhbGciOiJIUzI1NiIsImV4cCI6MTQ2NzM0MTQ3NCwiaWF0IjoxNDY3MzM3ODc0fQ.eyJpYXQiOjE0NjczMzc4NzQuNzE3MDYzLCJ1c2VyX2lkIjoiNDY1MDEyMjgzNDNiMTFlNmFhYTZhNDVlNjBlZDVlZDVmOTczYmEwZmNmNzgzYmI4YWRlMzRjN2I0OTJkOWU1NSIsInVzZXJfcm9sZSI6M30.23QD0OwLjdioKu5BgbaH2gHT2GoMz90n8VZcpvdyp7U
{% endhighlight %}
它是由“header.payload.signature”构成的。

### 3.2 如何解析token

解析需要使用到同样的serializer，配置一样的secret key和salt，使用loads方法来解析token。itsdangerous提供了各种异常处理类，用起来也很方便：
如果是SignatureExpired，则可以直接返回过期；
如果是BadSignature,则代表了所有其他签名错误的情况，于是又分为：
    能读取到payload：那么这个消息是一个内容被篡改、消息体加密过程正确的消息，secret key和salt很可能泄露了；
    不能读取到payload： 消息体直接被篡改，secret key和salt应该仍然安全。

以上内容写成一个函数，用于验证用户token。如果实现在python flask，可以考虑将此函数改为一个decorator修饰漆，将修饰器@到所有需要验证token的方法前面，则代码可以更加优雅。
{% highlight python %}
# serializer for JWT
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
# exceptions for JWT
from itsdangerous import SignatureExpired, BadSignature, BadData
    def tokenAuth(token):
        # token decoding
        s = Serializer(
            secret_key=api.app.config['SECRET_KEY'],
            salt=api.app.config['AUTH_SALT'])
        try:
            data = s.loads(token)
            # token decoding faild
            # if it happend a plenty of times, there might be someone
            # trying to attact your server, so it should be a warning.
        except SignatureExpired:
            msg = 'token expired'
            app.logger.warning(msg)
            return [None, None, msg]
        except BadSignature, e:
            encoded_payload = e.payload
            if encoded_payload is not None:
                try:
                    s.load_payload(encoded_payload)
                except BadData:
                    # the token is tampered.
                    msg = 'token tampered'
                    app.logger.warning(msg)
                    return [None, None, msg]
        if ('user_id' not in data) or ('user_role' not in data):
            msg = 'illegal payload inside'
            app.logger.warning(msg)
            return [None, None, msg]
        msg = 'user(' + data['user_id'] + ') logged in by token.'
#        app.logger.info(msg)
        userId = data['user_id']
        roleId = data['user_role']
        return [userId, roleId, msg]
{% endhighlight %}

## 4. 优化
上述的方法可以做到基本的JWT认证，但在实际开发过程中还有其他问题，于是各种借鉴，优化了一些东西。

token在生成之后，是靠expire使其过期失效的。签发之后的token，是无法收回修改的，因此涉及token的有效期的更改是个难题，它体现在以下两个问题：

* 问题1.用户登出

* 问题2.token自动延期

解决有效期问题，很自然能想到的是把每个token存库，设置一个valid字段，一旦注销了就valid=0；设置有效期字段，想要延期就增加有效期时间。openstack keystone就是这么做的。这个做法虽方便，但对数据库的压力较大，甚至在访问量较大，签发token较多的情况下，是对数据库的一个挑战。

在第三方认证协议Oauth2.0（<a href="https://tools.ietf.org/html/rfc6749">RFC6749</a>）则采取了另一种方法：refresh token。在用户首次认证后，签发两个token：

* 一个为access token，用于用户后续的各个请求中携带的人整信息

* 另一个是refresh token，为access token过期后，直接申请一个新的access token。

由此可以给两类不同token设置不同的有效期，例如给access token仅1小时的有效时间，而refresh token则可以是一个月。api的登出通过access token的过期来实现（前端则可直接抛弃此token实现登出），在refresh token的存续期内，访问api时可执refresh token申请新的access token（前端可存此refresh token，access token过其实进行更新，达到自动延期的效果）。

refresh token不可再延期，过期需重新使用用户名密码登录。

ps.前面提到创建token的时候将expire_in超时时间间隔作为函数的形参，是为了将此函数用于生成access token和refresh token，而两者的expire_in时间是不同的。

## 5. 总结一下

我们做了一个JWT的认证模块:
(access token在代码中为'token'，refresh token在代码中为'rftoken')

* 首次认证

client -----用户名密码----------->  server

client <------token、rftoken-----  server

* access token存续期内的请求

client ------请求（携带token）---->  server

client <-----结果-----------------  server

* access token超时

client ------请求（携带token）---->  server

client <-----msg:token expired---  server

* 重新申请access token

client -请求新token（携带rftoken）->  server

client <-----新token--------------  server

* rftoken token超时

client -请求新token（携带rftoken）->  server

client <----msg:rftoken expired---  server


如果设计一个针对此认证的前端，需要：

* 存储access token、refresh token

* 访问时携带access token，自动检查access token超时，超时则使用refresh token更新access token；状态延期用户无感知

* 用户登出直接抛弃access token与refresh token
