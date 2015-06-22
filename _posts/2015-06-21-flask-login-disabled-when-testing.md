---
layout: post
title: flask-login 在测试环境失效 
---

今天在运行flask的集成测试的时候发现，flask-login在测试环境下会失效。找了好久之后发现，
原来flask-login有个配置项`LOGIN_DISABLED`, 当这个配置不存在时，它的默认值取的是TESTING的值。
{% highlight python %}
 self._login_disabled = app.config.get('LOGIN_DISABLED',
                                        app.config.get('TESTING', False))
{% endhighlight %}

解决方法是在执行测试的时候，在flask的配置中添加 `LOGIN_DISABLED = True`。 
