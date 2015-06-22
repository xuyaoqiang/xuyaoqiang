---
layout: post
title: 在flask-login中同时使用不同的登录验证方式
---

flask-login 提供不同的登录验证方式，对于web环境可以使用 user_loader 的装饰器作为登录的验证函数, 它会有自动地帮你处理cookie：

{% highlight python %}
from flask.ext.login import LoginManager
login_manager = LoginManager()

@login_manager.user_loader
def load_user(userid):
    return User.get(userid)

{% endhighlight %}

flask-login 会将load_user返回的user对象赋给current_user这个全局对象

同样，当你是通过提供api的方式来提供服务的，不使用cookie，而是使用http的头部字段或者通过在请求url加入令牌的方式来进行用户验证，那user_loader就不太合适了，因为user_loader 的内部实现是通过读取cookie来实现的。这种情况下，  你就可以使用request_loader的装饰器来作为登录的验证函数.request_loader装饰器的返回值和user_loader一样，None或者user对象,一个例子:
{% highlight python %}
@login_manager.request_loader
def load_user_from_request(request):
    token = request.headers.get('UserToken')
    if token:
        user = User.query.filter(User.token == token ).first()
        return user 

    return None
{% endhighlight %}

注意这里的 load_user_from_request的参数request 就是flask中的全局对象, 它包含了一个http请求的全部属性和一些常用的方法, 比如请求的header、body、method等等。

这两种验证方式适用不同的业务环境，然而当某种环境同时需要两种登录验证方式怎么办?

比如当我们需要给移动端提供api服务时，通常我们都需要提供一个web端的admin后台给内容或者运营的伙伴使用，这个是我在工作中实际遇到的需求，这个时候我们该怎么办呢？

首先，我直接尝试提供了两种方式，然而flask-login 的加载user的方法是有先后顺序的，摘录一段flask-login的源码:

{% highlight python %}
 if is_missing_user_id:
     cookie_name = config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
     header_name = config.get('AUTH_HEADER_NAME', AUTH_HEADER_NAME)
     has_cookie = (cookie_name in request.cookies and
                   session.get('remember') != 'clear')
     if has_cookie:
         return self._load_from_cookie(request.cookies[cookie_name])
     elif self.request_callback:
         return self._load_from_request(request)
     elif header_name in request.headers:
         return self._load_from_header(request.headers[header_name])
{% endhighlight %}
也就是说flask-login会优先检查请求的cookie，当发现用户cookie不存在的时候才会根据request来验证，这显然不是我想要的，我希望能够根据请求来决定使用哪种方式来进行用户验证。
首先，我们将api和web管理后台分为两个不同的`blueprint`: `api` 和 `admin`。
然后我们利用一个全局变量 load_user_methods = {} 来保存不同blueprint的验证用户的函数:
{% highlight python %}
load_user_methods = {}

from api import api_blueprint 
load_user_methods[api_blueprint.name] = api_blueprint.load_user 
from admin import admin_blueprint
load_user_methods[admin_blueprint.name] = admin_blueprint.load_user

@login_manager.user_loader
def load_user(id):
    if request.blueprint in load_user_methods:
        method = load_user_methods.get(request.blueprint)
        return method(id)
    return None

{% endhighlight %}




