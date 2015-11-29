---
layout: post
title: RAML在RESTful API开发中的实践
---

[RAML](http://raml.org/)是一种基于YAML，用来描述RESTful API的设计语言。它可以很方便地表示RESTful API的CRUD的请求和响应格式，目前RAML的标准还在演进中，但是也已经有了很多非常棒的工具用来完善RESTful的设计和开发工作。

## 用RAML设计RESTful API
我们来设计一个电影的api `/movies`，
那么根据RESTful的风格，应该提供六个API接口
{% highlight python %}
GET /movies
POST /movies
GET /movies/{movieId}
PUT /movies/{movieId}
PATCH /movies/{movieId}
DELETE /movies/{movieId}
{% endhighlight %}
那么用RAML表示的`/movies` 设计应该是:

{% highlight python %}
#%RAML 0.8

title: Movie API
baseUri: http://example.api.com/{version}
version: v1
traits:
  - paged:
      queryParameters:
        pages:
          description: The number of pages to return
          type: number
  - secured: !include http://raml-example.com/secured.yml
/movies:
  is: [ paged, secured ]
  get:
  post:
  /{movieId}:
    get:
      response:
        200:
          body:
            application/json:
              schema:
        
    put:
    patch:
    delete:
      description: |
        This method will *delete* an **individual movie**

{% endhighlight %}


假设每部电影都包含以下的属性:
{% highlight python %}
id: - id
name: - 电影名称
director: - 导演
actors: - 演员
movies_type: - 类型
country: - 国家地区
{% endhighlight %}


