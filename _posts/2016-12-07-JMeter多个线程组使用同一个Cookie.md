---
layout: post
title: JMeter多个线程组使用同一个Cookie
date: 2016-12-07 09:45
disqus: y
---

**测试场景：**  
多个硬件设备客户端，第一次启动都会用同一个账号登录，之后定时轮询多个服务器接口。  

**问题：**  
脚本里每个线程代表一台设备，如果每个线程都登录一次，不符合真实场景。  
这样需要线程组 A 登录一次，其他线程组在 A 执行成功后再执行，并使用 A 获得 Cookie。  
开始把 HTTP Cookie Manager 放在顶层测试计划下，发现其他线程组没有没有取到A得到的 Cookie。  
搜索后发现，JMeter 每个线程都有自己的“Cookie 存储区域”  
[JMeter文档](http://jmeter.apache.org/usermanual/component_reference.html)  
> First, it stores and sends cookies just like a web browser. If you have an HTTP Request and the response contains a cookie, the Cookie Manager automatically stores that cookie and will use it for all future requests to that particular web site. Each JMeter thread has its own "cookie storage area". So, if you are testing a web site that uses a cookie for storing session information, each JMeter thread will have its own session. Note that such cookies do not appear on the Cookie Manager display, but they can be seen using the View Results Tree Listener.  

**一个解决方案：**  
测试计划下添加 HTTP Header Manager，添加键值对`Cookie：session_id=${__property(storeid)}`  
线程组 A 登录 Sample 下添加 BeanShell PostProcessor，Script 填写`${__setProperty(storeid, ${ex_session_id})};`