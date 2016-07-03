---
layout: post
title: 命令行启动 AppiumForWindows
date: 2015-08-13 09:44
disqus: y
---

比如安装在D:\Program Files\Appium，cmd执行
`"D:\Program Files\Appium\node.exe"  "D:\Program Files\Appium\node_modules\appium\bin\appium.js"`
启动成功
>info: Welcome to Appium v1.3.4 (REV c8c79a85fbd6870cd6fc3d66d038a115ebe22efe)
info: Appium REST http interface listener started on 0.0.0.0:4723
info: Console LogLevel: debug

加上参数
`"D:\Program Files\Appium\node.exe"  "D:\Program Files\Appium\node_modules\appium\bin\appium.js" -p 4724`
参数生效
>info: Welcome to Appium v1.3.4 (REV c8c79a85fbd6870cd6fc3d66d038a115ebe22efe)
info: Appium REST http interface listener started on 0.0.0.0:4724
info: [debug] Non-default server args: {"port":4724}
info: Console LogLevel: debug

在D:\Program Files\Appium\写好的批处理appium.cmd，内容是
```cmd
@IF EXIST "%~dp0\node.exe" (
  "%~dp0\node.exe"  "%~dp0\node_modules\appium\bin\appium.js" %*
) ELSE (
  @SETLOCAL
  @SET PATHEXT=%PATHEXT:;.JS;=;%
  node  "%~dp0\node_modules\appium\bin\appium.js" %*
)
```

然后把路径添加到环境变量Path中，cmd执行
`appium.cmd`
启动成功
>info: Welcome to Appium v1.3.4 (REV c8c79a85fbd6870cd6fc3d66d038a115ebe22efe)
info: Appium REST http interface listener started on 0.0.0.0:4723
info: Console LogLevel: debug
