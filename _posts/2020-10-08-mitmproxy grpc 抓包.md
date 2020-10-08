---
layout: post
title: mitmproxy grpc 抓包 
date: 2020-10-08 10:08
disqus: y
---

[mitmproxy](https://mitmproxy.org) 5.2 支持了 http trailer，可以正常代理 grpc 请求了。
grpc 原始信息不可读，可以写个[自定义视图](https://docs.mitmproxy.org/stable/addons-examples/#contentview)显示反序列化后的内容。

先用 grpc [官方示例](https://github.com/grpc/grpc/blob/master/examples/python/helloworld/greeter_server.py)启动一个 grpc 服务：  

```python
python greeter_server.py
```

客户端就用 [bloomrpc](https://github.com/uw-labs/bloomrpc)，不另外写代码了。导入对应的 [proto](https://github.com/grpc/grpc/blob/master/examples/protos/helloworld.proto) 文件，向 localhost:50051 发送请求验证服务有没有正常工作：
![](https://testerhome.com/uploads/photo/2020/13b38014-2714-45dd-9089-40c4dd5cdcce.png!large)

因为 mitmproxy 目前[只支持 TLS 上的 HTTP/2](https://github.com/mitmproxy/mitmproxy/issues/3362)，还需要再用 nginx 转发并加上 TLS。
参考这篇[文章](https://gist.github.com/fntlnz/cf14feb5a46b2eda428e000157447309)生成自签名证书： 

```bash
openssl genrsa -des3 -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
openssl genrsa -out localhost.key 2048
openssl req -new -key localhost.key -out localhost.csr # Common Name 输入域名 localhost，其他随便填一下
openssl req -in localhost.csr -noout -text
openssl x509 -req -in localhost.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out localhost.crt -days 500 -sha256
openssl x509 -in localhost.crt -text -noout
cat localhost.crt localhost.key > localhost.pem
```

配置 nginx：  

```
server {
    listen 6556 ssl http2;
    server_name  localhost;

    ssl_certificate ~/localhost.pem;
    ssl_certificate_key ~/localhost.pem;

    location / {
        grpc_pass grpc://localhost:50051;
    }
}
```

配置好启动 nginx，在 bloomrpc 里导入证书：
![](https://testerhome.com/uploads/photo/2020/7166a204-db58-4b35-b169-b5e2050b72e1.png!large)


向 localhost:6556 发送请求验证 nginx 有没有正常工作：
![](https://testerhome.com/uploads/photo/2020/70308096-cd1c-4068-9570-8f56d74c1511.png!large)

启动 mitmproxy，因为是自签名证书，多加两个参数：  

```bash
mitmweb --certs localhost=~/localhost.pem -k
```

MacOS 上，通过下面的命令重新启动 bloomrpc，可以让 bloomrpc 发出的请求经过代理：  

```bash
export http_proxy=http://127.0.0.1:8080;export https_proxy=http://127.0.0.1:8080;
open /Applications/BloomRPC.app
```

再次向 localhost:6556 发送请求，验证 mitmweb 有没有正常工作：
![](https://testerhome.com/uploads/photo/2020/0c5a1241-ba90-4a88-9778-49eff77bd390.png!large)


然后写自定义视图，先安装 mitmproxy 依赖： 

```bash
pip install mitmproxy
```

新建 `grpc_addon.py` 内容如下：  

```python
from mitmproxy import contentviews, http
import helloworld_pb2

class GrpcView(contentviews.View):
    name = 'grpc'

    def __call__(self, data, **metadata) -> contentviews.TViewResult:
        # 参考 https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md，grpc 请求 body 的第一个字节代表有没有被压缩，示例中没有压缩，所以不需要额外处理。二到五字节是 protobuf 内容的长度。
        compressed_flag = data[0]
        message_length = int.from_bytes(data[1:5], 'big')
        message = data[5:message_length + 5]
        # 之后就是反序列化 protobuf
        if isinstance(metadata['message'], http.HTTPRequest):
            message_method = helloworld_pb2.HelloRequest
        elif isinstance(metadata['message'], http.HTTPResponse):
            message_method = helloworld_pb2.HelloReply
            
        target = message_method()
        target.ParseFromString(message)
        
        return 'deserialized grpc', contentviews.format_text(str(target))

grpc_view = GrpcView()

def load(l):
    contentviews.add(grpc_view)

def done():
    contentviews.remove(grpc_view)
```

重新启动 mitmproxy，加载 `grpc_addon.py`：  

```bash
mitmweb --certs localhost=~/localhost.pem -k -s grpc_addon.py
```

再次向 localhost:6556 发送请求，可以查看反序列化后的内容了：
![](https://testerhome.com/uploads/photo/2020/cd6bb09b-e404-4ff1-8c6e-795c53a208de.png!large)

也可以改写响应内容，在 `grpc_addon.py` 加上如下内容后再发送请求看看：  

```python
class Rewrite:
    def response(self, flow):
        data = flow.response.raw_content
        compressed_flag = data[0]
        message_length = int.from_bytes(data[1:5], 'big')
        message = data[5:message_length + 5]
        target = helloworld_pb2.HelloReply()
        target.ParseFromString(message)
        target.message = 'Hello, rewrited'
        modified_message = target.SerializeToString()
        # 改写后内容长度可能会变，所以要重新计算长度
        modified_response = data[0].to_bytes(1, byteorder='big') + len(modified_message).to_bytes(4, byteorder='big') + modified_message
        flow.response.headers['content-length'] = str(len(modified_response))
        flow.response.raw_content = modified_response

addons = [
    Rewrite()
]
```