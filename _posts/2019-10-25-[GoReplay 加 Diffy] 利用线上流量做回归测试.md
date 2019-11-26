---
layout: post
title: 【GoReplay 加 Diffy] 利用线上流量做回归测试
date: 2019-10-25 19:20
disqus: y
---

[GoReplay 项目](https://github.com/buger/goreplay)  
[Diffy 项目](https://github.com/opendiffy/diffy)  

对于没有副作用的接口（重复发送不会产生两份数据、不会产生多余的监控统计等等），就可以用这种方式方便的做回归测试。  
部署三个不接外部流量的服务，两份老版本、一份新版本，把生产环境的流量复制到 Diffy 上。  
如果生产环境支持通过请求头之类的方式区分测试流量和真实流量，就可以扩大使用范围。  
![](https://testerhome.com/uploads/photo/2019/1956de79-336e-47b2-9184-8ffcc98357f6.png!large)  

本机启动一个服务 serverA，监听 8000 端口，访问 `127.0.0.1:8000/testerhome` 会返回一个 JSON  

```bash
docker run --detach --publish=8000:80 --name=serverA nginx
docker exec -it serverA bash
apt update
apt install vim
vim /etc/nginx/conf.d/default.conf
# 添加一个 Path
location /testerhome {
      default_type application/json;
      return 200 '{"tag":"old","noise":"1"}';
}
# :wq 保存退出，重载 Nginx
nginx -s reload
# 退出容器
exit
# 输出 Nginx log 到终端上
docker logs -f serverA
```
访问 http://127.0.0.1:8000/testerhome  

```bash
curl http://127.0.0.1:8000/testerhome 
```
可以看到返回了 `{"tag":"old","noise":"1"}`，终端也输出了如下日志  
> 172.17.0.1 - - [13/Oct/2019:07:04:04 +0000] "GET /testerhome HTTP/1.1" 200 28 "-" "curl/7.54.0" "-" "-" 

再打开一个终端，启动一个服务 serverB，监听 8010 端口，配置同样的 Path  和响应  

```bash
docker run --detach --publish=8010:80 --name=serverB nginx
docker exec -it serverB bash
apt update
apt install vim
vim /etc/nginx/conf.d/default.conf
# 添加一个 Path
location /testerhome {
      default_type application/json;
      return 200 '{"tag":"old","noise":"1"}';
}
# :wq 保存退出，重载 Nginx
nginx -s reload
# 退出容器
exit
# 输出 Nginx log 到终端上
docker logs -f serverB
```

再打开一个终端，下载解压 gor 后启动，将 8000 端口监听到的请求复制一份发送到 8010 端口  

```bash
curl -L -O https://github.com/buger/goreplay/releases/download/v1.0.0/gor_1.0.0_x64.tar.gz
tar xvzf gor_1.0.0_x64.tar.gz
sudo ./gor --input-raw :8000 --output-http http://127.0.0.1:8010
```

访问 http://127.0.0.1:8000/testerhome  

```bash
curl http://127.0.0.1:8000/testerhome 
```
可以看到 8010 端口上的 serverB 也产生了访问日志  

> 172.17.0.1 - - [13/Oct/2019:07:14:04 +0000] "GET /testerhome HTTP/1.1" 200 28 "-" "curl/7.54.0" "-" "-" 

再打开一个终端，启动一个服务 serverC，监听 8020 端口，配置同样的 Path ，响应是 `{"tag":"old","noise":"2"}`  

```bash
docker run --detach --publish=8020:80 --name=serverC nginx
docker exec -it serverC bash
apt update
apt install vim
vim /etc/nginx/conf.d/default.conf
# 添加一个 Path
location /testerhome {
      default_type application/json;
      return 200 '{"tag":"old","noise":"2"}';
}
# :wq 保存退出，重载 Nginx
nginx -s reload
# 退出容器
exit
# 输出 Nginx log 到终端上
docker logs -f serverC
```

再打开一个终端，启动一个服务 serverD，监听 8030 端口，配置同样的 Path ，响应是 `{"tag":"new","noise":"3"}`  

```bash
docker run --detach --publish=8030:80 --name=serverD nginx
docker exec -it serverD bash
apt update
apt install vim
vim /etc/nginx/conf.d/default.conf
# 添加一个 Path
location /testerhome {
      default_type application/json;
      return 200 '{"tag":"new","noise":"3"}';
}
# :wq 保存退出，重载 Nginx
nginx -s reload
# 退出容器
exit
# 输出 Nginx log 到终端上
docker logs -f serverD
```

再打开一个终端，[下载](https://github.com/opendiffy/diffy/releases)或自己编译 Diffy 并启动  

```bash
# 编译 Diffy 需要 jdk8，注意要用 opendiffy/diffy，和 twitter/diffy 不一样
git clone https://github.com/opendiffy/diffy.git
cd diffy
./sbt assembly
# 最终会生成 ./target/scala-2.12/diffy-server.jar，启动它
java -jar ./target/scala-2.12/diffy-server.jar \
-candidate=localhost:8030 \
-master.primary=localhost:8010 \
-master.secondary=localhost:8020 \
-service.protocol=http \
-serviceName=My-Service \
-proxy.port=:8880 \
-admin.port=:8881 \
-http.port=:8889 \
-rootUrl='localhost:8889' \
-summary.email="happy@a.com"
```

浏览器打开 http://127.0.0.1:8889，看到展示 Diffy 结果的界面  
![](https://testerhome.com/uploads/photo/2019/0b2cc07b-2cf4-4197-b4fe-db891e469362.png!large)


重启 gor，这次把将 8000 端口监听到的请求复制一份转发到  Diffy 的 8880 端口  

```bash
sudo ./gor --input-raw :8000 --output-http http://127.0.0.1:8880
```

访问http://127.0.0.1:8000/testerhome ，可以看到 8010、8020、8030 端口上的三个服务都产生了访问日志  

```bash
curl http://127.0.0.1:8000/testerhome 
```

Diffy 上出现了对比结果：  
![](https://testerhome.com/uploads/photo/2019/ad31b231-47d8-4372-a6e6-b181be189e8b.png!large)

如果一个字段在 master.primary 和 master.secondary 上不一致，很有可能不是 bug，时间戳或者个性推荐之类的数据会这样。这时候可以 把 Exclude Noise 开关打开，排除这些“噪声”  
![](https://testerhome.com/uploads/photo/2019/92a93f08-90f2-4d8c-bb3f-5db6f1dfa8c7.png!large)
