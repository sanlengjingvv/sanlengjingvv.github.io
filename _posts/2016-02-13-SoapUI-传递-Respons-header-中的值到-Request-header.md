---
layout: post
title: SoapUI-传递 Respons header 中的值到 Request header
date: 2016-02-13 09:45
disqus: y
---


SoapUI 的 Property Transfer 不能在请求头和响应头之前传递参数，查到可以用 [Groovy Script](http://www.soapui.org/apidocs/index.html) 在信息头间传参。

比如登录请求是：
```
http://10.0.0.1/mobile/login/?username=13740434043&password=123456
```

响应是：
```
HTTP/1.1 200 OK
Server: nginx/1.6.2
……
Set-Cookie: frontend=bse0s06ef9sd65k8ho5ah9ecm1; expires=Wed, 04-Nov-2015 09:25:14 GMT; path=/; domain=10.0.0.1; HttpOnly
……

<message>
   <status>success</status>
   <text>登录成功/text>
</message>
```

需要将登录响应头中的 Set-Cookie 的值传递到“获取订单列表”请求头的 cookie 中：
```
GET http://10.0.0.1/mobile/orderlist/ HTTP/1.1
……
cookie: frontend=bse0s06ef9sd65k8ho5ah9ecm1
……
```

在两个请求间插入 Groovy Script
![插入 Groovy Script](http://ww4.sinaimg.cn/large/4835282djw9exp6c8s9tcj206q05cq34.jpg)

代码如下：
```Groovy
def responseCookie = testRunner.testCase.getTestStepByName("login").httpRequest.response.responseHeaders["Set-Cookie"]
def frontend = (responseCookie =~ "frontend=\\w{26}")[0]   //正则表达式截取需要的部分
def orderHeaders = testRunner.testCase.testSteps["order"].getHttpRequest().getRequestHeaders()
def list = []
list.add(frontend)
orderHeaders["cookie"] = list;
testRunner.testCase.testSteps["order"].getHttpRequest().setRequestHeaders(orderHeaders)
```
