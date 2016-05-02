---
layout: post
title: X-XSS-Protection引发的base标签失效问题
---

## 问题背景

  运营的同学有一天反映管理后台的一个表单，提交之后，突然页面上的css和js文件都load不出来，而正常的访问url页面则一切正常。查看页面的源码后发现，之前设置的 \<base href="/"> 失效了。
  
## 查找过程
查找问题的原因，首先列举所有可能出错的环节：

> 1. 提交后服务器逻辑修改了页面。
> 2. 引入的一些js代码里可能修改了base标签。
> 3. chrome的设置导致了base标签失败。

  在查看了服务器的代码后发现，服务端在表单提交完成后，只是做了一个重定向到当前页面的操作，并不会修改页面的代码，那么 `1` 排除。

  至于 `2` 则更加不可能了，因为`base` 标签的失效，导致页面上所有的js文件都请求到了相对地址，js文件都出现了404错误。js文件连下载都不成功，更不可能执行了。

  那文件的罪魁应该是浏览器(chrome)了。于是同样的操作，我在safari上执行了一遍，ok了！
  
  越是灵机一动，这种错误，也许chrome会给一些错误提示。于是点开chrome的检查后台，果然，出现了一个错误：

> The XSS Auditor refused to execute a script because 
> its source code was
> found within the request. The auditor was enabled as the server sent  
> neither an 'X-XSS-Protection' nor 'Content-Security-Policy' header.

查阅了相关资料后发现，这是chrome防止xss跨站脚本攻击添加的安全策略，当`提交的request和返回的response中包含同样的js代码`时会触发，导致`base`标签的href没有被设置。

## 解决办法

既然找到原因了，那么解决办法也很简单：

1. 去掉提交的表单中的js代码。
2. 如果表单中的js代码无法去掉，则可以设置http 返回的响应header

>  X-XSS-Protection: 0

因为页面是需要登录的管理后台，所以disable掉这个策略不会影响后台的安全性。对于其他面向用户的站点，还是不建议使用这种方法。

flask的后台可以这样写：

	@app.after_request
	def add_headers(response):
		response.headers.extend({"X-XSS-Protection": 0})
		return response

## 总结

有句名言说的好：排除掉所有不可能的因素，剩下的即使再不可思议，也是真实的答案！

当程序出现问题时，我们可以列举所有可能出错的环节，然后一个个分析排除，总能找到问题的现场和原因。





