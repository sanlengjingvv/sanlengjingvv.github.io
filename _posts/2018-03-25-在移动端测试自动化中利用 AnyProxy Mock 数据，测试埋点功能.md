---
layout: post
title: 在移动端测试自动化中利用 AnyProxy Mock 数据，测试埋点功能 
date: 2018-03-25 19:20
disqus: y
---

[本帖使用的示例代码在这里](https://github.com/sanlengjingvv/anyproxy-automation-example)  
**20180705更新**：发现不需要使用数据库中转数据，所以在 master 分支去掉了 SQLite 和 Sequelize，老代码在 using-SQLite 分支  

### Mock 数据  
以 TesterHome iOS 客户端为例，一个测试点是：在话题列表，很长的标题能不能正确展示。  
如果组织分工上移动端团队是分离的，或者功能上这个列表是智能推荐，或者技术上测试环境治理水平不高接口总是不可用，测试和开发同学在测试这点时最常见的选择是通过代理工具将标题改长，观察 App 的展示效果，延后对服务端的依赖。  
比如 TesterHome 客户端里，启动后首屏请求
`https://testerhome.com/api/v3/topics.json?limit=40&offset=0&type=last_actived`
获取话题列表，把 title 改长再返回给客户端：  

```json
{
	"topics": [{
		"id": 12390,
		"title": "职业这条不归路上！总监们帮你解惑",
		"replies_count": 21,
		"node_name": "活动沙龙",
		"user": {
			"id": 3903,
			"abilities": {
				"update": false,
			}
		},
		"abilities": {
			"update": false
		}
	}]
}
```

想在 UI 自动化时也使用这个方式，需要代理工具支持可编程的方式编写规则、和外部交互，比如 [AnyProxy](http://anyproxy.io/cn/) 。  
替换标题的规则文件如下：  

```javascript
module.exports = {
    *beforeSendResponse(requestDetail, responseDetail) {
        // 这个请求响应 Code 可能是 304
        if ((requestDetail.url.indexOf('https://testerhome.com/api/v3/topics.json?limit=40&offset=0&type=last_actived') === 0) && responseDetail.response.statusCode === 200) {
            let newResponse = responseDetail.response
            let jsonBody = JSON.parse(newResponse.body.toString())
            jsonBody.topics[1].title = '很长很长的标题显示测试' + Date.now()
            newResponse.body = JSON.stringify(jsonBody)
            return {
                response: newResponse
            }
        }
    }
}
```
如果使用 Appium ，为了方便和 AnProxy 交互，可以选一个 JavaScript 实现的客户端，比如 [WebdriverIO](http://webdriver.io) 。  
在 AnyProxy 的规则文件里声明[引用类型](http://hellobug.github.io/blog/javascript-variable-assignment/)的变量，在 WebdriverIO 的用例中给这个变量赋值，这样不同用例使用不同值的时候不需要修改 AnyProxy 规则文件。
于是 AnyProxy 规则文件变成这样：  

```javascript
let testData = {}

module.exports = {
    testData: testData,
    *beforeSendResponse(requestDetail, responseDetail) {
        // 这个请求响应 Code 可能是 304
        if ((requestDetail.url.indexOf('https://testerhome.com/api/v3/topics.json?limit=40&offset=0&type=last_actived') === 0) && responseDetail.response.statusCode === 200) {
            let newResponse = responseDetail.response
            let jsonBody = JSON.parse(newResponse.body.toString())
            if (Object.keys(testData).length !== 0) {
                jsonBody.topics[testData.index].title = testData.title
            }
            newResponse.body = JSON.stringify(jsonBody)
            return {
                response: newResponse
            }
        }
    }
}
```
WebdriverIO 用例这样写：  

```javascript
const assert = require('assert')
let { testData } = require('../rules') //引入 AnyProxy 规则文件

describe('TesterHome', function () {
    it('修改数据', function () {
        testData.index = 1
        testData.title = '很长很长的标题显示测试' + Date.now()
        
        // Appium 在建立 session 后就会启动 App，这时候 testData 还没被赋值，所以在赋值后重启 App 重新获取话题列表数据
        browser.closeApp()
        browser.launch()
        browser.pause(3000)
        
        // 话题列表话题的 Xpath 序号从 1 开始，但 HTTP 响应的 JSON 是从 0 开始
        const listIndex = testData.index + 1
        const locator = `//XCUIElementTypeTable/XCUIElementTypeCell[${listIndex}]/XCUIElementTypeStaticText`
        assert.equal($(locator).getText(), testData.title)
    })
})
```

完整的测试执行过程是这样的：  
1、 获取执行机 IP 。  
2、 iOS 使用 Appium(WebDriverAgent)，通过 UI 自动化的方式打开“设置”修改代理，Android 通过 ADB 修改代理，比如用这个 [AndroidProxySetter](https://github.com/jpkrause/AndroidProxySetter) 。  
3、 启动 WebdriverIO 测试，通过 WebdriverIO 的钩子，在 Appium 建立 session 前启动 AnyProxy 。  

```javascript
// wdio.conf.js

let rules = require('./rules') //引入 AnyProxy 规则文件

// AnyProxy 启动参数
const options = {
    port: 8001,
    rule: rules,
    webInterface: {
        enable: true,
        webPort: 8002,
        wsPort: 8003,
    },
    throttle: 10000,
    forceProxyHttps: true,
    silent: true
}
const proxyServer = new AnyProxy.ProxyServer(options)
proxyServer.on('ready', () => { console.log("anyproxy ready") })
proxyServer.on('error', (e) => { console.log("anyproxy error") })

exports.config = {
    beforeSession: function (config, capabilities, specs) {
        // 启动 AnyProxy
        proxyServer.start()
    }
}
```
4、 进入测试用例，给需要的、代表测试数据的变量赋值。  
5、 重启 App，这时候 App 获取的数据会被 AnyProxy 改为前一步赋予的值。  
6、 验证 App 功能是否正确。  
7、 通过 WebdriverIO 的钩子，在 Appium 关闭 session 后停止 AnyProxy 。  

```javascript
// wdio.conf.js

exports.config = {
    afterSession: function (config, capabilities, specs) {
        // 停止 AnyProxy
        proxyServer.close()
    }
}
```
8、 恢复手机为没有使用代理的状态。  

### 测试埋点功能  
TesterHome 用了 google-analytics 做统计，启动 App 进入首页会触发一个 GET 请求
`https://www.google-analytics.com/collect?v=1&_v=j66&a=18908429&t=pageview&_s=1&dl=https%3A%2F%2Ftesterhome.com%2Ftopics%3Faccess_token%3Dfa09aacd46441b55e6d64d04983798edfc2914d024d62cf2d469ff754193592f&ul=zh-cn&de=UTF-8&dt=%E7%A4%BE%E5%8C%BA%20%C2%B7%20TesterHome&sd=32-bit&sr=414x736&vp=100x56&je=0&_u=ACCAgEQ~&jid=147019750&gjid=698481509&cid=1198303934.1481097811&tid=UA-45014075-1&_gid=666582918.1521891128&z=349772242`
URL decode 之后是：
`https://www.google-analytics.com/collect?v=1&_v=j66&a=18908429&t=pageview&_s=1&dl=https://testerhome.com/topics?access_token=fa09aacd46441b55e6d64d04983798edfc2914d024d62cf2d469ff754193592f&ul=zh-cn&de=UTF-8&dt=社区 · TesterHome&sd=32-bit&sr=414x736&vp=100x56&je=0&_u=ACCAgEQ~&jid=147019750&gjid=698481509&cid=1198303934.1481097811&tid=UA-45014075-1&_gid=666582918.1521891128&z=349772242`

如果使用 AnyProxy ，发现这个请求的时候把这些数据持久化，比如存入本地的 SQLite 数据库，就可以在测试用例中去验证了。
AnyProxy 规则文件如下，这里使用了 [Sequelize](http://docs.sequelizejs.com) 这个 Node.js 的 ORM 库。  

```javascript
const querystring = require('querystring')
const fs = require('fs')
const Sequelize = require('sequelize')

// 每次启动 AnyProxy 的时候使用新的 SQLite 数据库
const databaseFile = './database.sqlite'
if (fs.existsSync(databaseFile)) {
    console.log('Removing ' + databaseFile)
    fs.unlinkSync(databaseFile)
  }

// sequelize 配置信息
const sequelize = new Sequelize('mainDB', null, null, {
    host: 'localhost',
    dialect: 'sqlite',
    logging: false,

    pool: {
        max: 15,
        min: 0,
        acquire: 30000,
        idle: 10000
    },

    storage: databaseFile
})

// 建立数据库连接
sequelize.authenticate().then(() => {
    console.log('Database connection has been established successfully.')
}).catch(err => { 
    console.error('Unable to connect to the database:', err)
})

// 定义 Model，表示 www.google-analytics.com/collect 请求中 query string 和表结构的对应关系
const Collect = sequelize.define('collect', {
    id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    t: {
        type: Sequelize.STRING
    },
    dt: {
        type: Sequelize.STRING
    }
})

module.exports = {
    *beforeSendRequest(requestDetail) {
        if (requestDetail.url.indexOf('https://www.google-analytics.com/collect') === 0) {
            const collect = querystring.parse(requestDetail.requestOptions.path)
            // 将埋点上报数据存入数据库
            Collect.sync({ force: false }).then(() => {
                return Collect.create(collect).then(() => {
                    return null
                }).catch(error => {
                    console.log('[Database] error' + error)
                })
            })
        }
    }
}
```

使用 WebdriverIO 时，测试用例如下：  
  
```javascript
const assert = require('assert')
const { Collect } = require('../database')

describe('TesterHome', function () {
    it('验证埋点', function () {
        const expectDt = '社区 · TesterHome'
        const expectT = 'pageview'

        browser.pause(3000)

        return Promise.all([
            // 查询数据库，取出数据后断言
            Collect.findOne({ where: { dt: expectDt } }).then(collect => {
                assert.equal(collect.t, expectT)
            })
        ])
    })
})
```

2019.10.10 更新：[在每一个自动化测试用例里建立代理规则 ](https://testerhome.com/topics/20772)  