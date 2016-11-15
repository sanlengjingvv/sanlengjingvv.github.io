---
layout: post
title: Burp Suite & Charles HTTPS抓包
date: 2016-02-13 09:45
disqus: y
---

### 准备：

先看这篇文章，[Nginx 配置 SSL 证书 + 搭建 HTTPS 网站教程](https://s.how/nginx-ssl/)
你会有配套的两个文件，一个后缀名`.key`另一个`.crt`
比如`testerhome.key`和`testerhome.crt`吧

### 生成p12文件  
  
```bash
testerdeMac:forNginx tester$ pwd
/Users/tester/Downloads/forNginx
testerdeMac:forNginx tester$ ls
testerhome.crt	testerhome.key
testerdeMac:forNginx tester$ openssl pkcs12 -export -clcerts -in testerhome.crt -inkey testerhome.key -out testerhome.p12
Enter Export Password:
Verifying - Enter Export Password:
```
  
- OS X 和 Linux 已经安装了 OpenSSL ，可以直接执行`openssl`命令  
- 设个密码，后面会用到，比如密码是`testerhome`  
- 引用自[cer证书转换成为p12证书的方法](http://www.doc88.com/p-9039007440454.html)  
  
### Burp Suite设置  
启动 [Burp Suite](https://portswigger.net/)  
`java -jar burpsuite_free_v1.6.32.jar`  

修改代理设置  
![Burp Suite设置](/images/2016/9dfcc754d1bb4a59ff633e8e9102c01f.png)  


### Charles 设置  
版本：Charles 3.11.2  
菜单 -> Proxy -> SSL Proxying Settings  
![CharlesSSL](/images/2016/2b4e6c1ba6ea6007c5ef5ce2d8934446.png)  
![CharleClient](/images/2016/c81679c676f78e27c32420925588a391.png)  
![CharleRoot](/images/2016/5fa280772858c30e05feed4bdd62dd94.png)  
完成就可以抓到HTTPS了  