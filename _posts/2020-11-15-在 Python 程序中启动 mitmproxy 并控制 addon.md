---
layout: post
title: 在 Python 程序中启动 mitmproxy 并控制 addon.md
date: 2020-11-15 20:08
disqus: y
---

mitmproxy 本身是个 Python 包，比如`mitmweb`命令实际是如下代码：  
```bash
#!/Users/niconi/venv/bin/python3
# -*- coding: utf-8 -*-
import re
import sys
from mitmproxy.tools.main import mitmweb
if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(mitmweb())
```

mitmproxy 提供了 [addon](https://docs.mitmproxy.org/stable/addons-overview/) 的方式编写代理规则，比如如下 addon 给每个请求加上一个请求头 `mitmproxy:default`：  
```python
# addon.py
from mitmproxy import contentviews, http

class Rewrite:
    flag = 'default'

    @classmethod
    def set_flag(self, value):
        self.flag = value


    def request(self, flow):
        flow.request.headers['mitmproxy'] = self.flag


addons = [
    Rewrite()
]
```

如果只是想在一个 Python 程序中启动 mitmproxy，可以用 `subprocess`，但因为是另一个内存空间，很难做到在程序运行时控制、修改 addon，用线程可以达到这个效果：  

```python
# mitmweb_wraper.py
from mitmproxy import proxy, options
from mitmproxy.tools.web.master import WebMaster
import threading,asyncio

import addon


class MitmWebWraper(object):
    mitmweb = None
    thread = None

    def loop_in_thread(self, loop, mitmweb):
        asyncio.set_event_loop(loop)
        mitmweb.run()

    def start(self):
        opts = options.Options(listen_host='0.0.0.0', listen_port=8080,
                               confdir='~/.mitmproxy', ssl_insecure=True)
        pconf = proxy.config.ProxyConfig(opts)
        self.mitmweb = WebMaster(opts)
        self.mitmweb.server = proxy.server.ProxyServer(pconf)
        self.mitmweb.addons.add(addon)
        loop = asyncio.get_event_loop()
        self.thread = threading.Thread(
            target=self.loop_in_thread, args=(loop, self.mitmweb))

        try:
            self.thread.start()
        except KeyboardInterrupt:
            self.mitmweb.shutdown()


if __name__ == "__main__":
    mitmweb_wraper = MitmWebWraper()
    mitmweb_wraper.start()
```

在另一个 Python 程序中，通过 mitmweb_wraper 启动 mitmweb，并且 import addon，对 addon 的修改会立即生效，新增的请求头会变成 `mitmproxy:changed`：  
```python
import mitmweb_wraper, addon

mitmweb_wraper = mitmweb_wraper.MitmWebWraper()
mitmweb_wraper.start()

addon.Rewrite().set_flag('changed')
```