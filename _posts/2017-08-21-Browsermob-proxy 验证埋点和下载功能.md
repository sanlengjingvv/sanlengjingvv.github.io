---
layout: post
title: Browsermob-proxy 验证埋点和下载功能
date: 2017-08-21 19:20
disqus: y
---

[Browsermob-proxy 项目地址](https://github.com/lightbody/browsermob-proxy)  

**测试场景说明：**  
场景A：  
在报表界面，类型选择“日报”，日期选“2016-08-03”，点“下载”，下载一份 csv 格式的文件。  
请求：  

```
GET /report?type=day&date=2016-08-03 HTTP/1.1
Host: testerhome.com
...
```
响应：  

```
HTTP/1.1 200 OK
Content-Type: text/csv

日期,销售额
2016-08-03,3210.40
```
对于`Content-Type: text/csv`，把 body 保存为 csv 文件是浏览器行为，所以在对被测系统，可以把点“下载”后的请求作为验证点。  

场景B：  
统计网页中推荐位的展示量。  
1、打开网页，获取推荐位内容：  

```
GET /recommend HTTP/1.1
Host: testerhome.com
...
```

```
HTTP/1.1 200 OK
Content-Type: application/json

{
    "content-id": 508,
    "content": "TestHome",
    "url": "http://testerhome.com/"
}
```
2、生成推荐位部分的 HTML 如下：  

```HTML
<div id="recommend">
    <a href="http://testerhome.com/" content-id="508">TestHome</a>
</div>
```
3、推荐位被展示时，发出包含统计信息的异步请求：  

```
POST /statistics HTTP/1.1
Host: testerhome.com
Content-Type: application/json

{
    "content-id": 508,
    "event": "show"
}
```
```
HTTP/1.1 200 OK
Content-Type: application/json

{
    "status": "success"
}
```

这里把滚动到`id="recommend"`元素后发出的请求作为验证点。  

**代码示例：**   
1、这里用了 [Selenide](http://selenide.org/)  和 [AssertJ](http://joel-costigliola.github.io/assertj/)。  
2、Browsermob-proxy 可以将捕获的 HTTP 内容保存为 [HAR](http://www.softwareishard.com/blog/har-12-spec/) 格式，也是 Json ，就用 JsonPath 查找需要的内容了。  

下载报表：  

```java
public class Report {
    BrowserMobProxy proxy;

    @BeforeMethod
    public void beforeMethod() {
        com.codeborne.selenide.Configuration.browser="chrome";
        proxy = new BrowserMobProxyServer();
        proxy.start(0);
        Proxy seleniumProxy = ClientUtil.createSeleniumProxy(proxy);
        WebDriverRunner.closeWebDriver(); //注意，需要在开启 Driver 前设置代理，需要根据自己用例组织情况决定是否重启 Driver
        WebDriverRunner.setProxy(seleniumProxy);
        proxy.enableHarCaptureTypes(CaptureType.REQUEST_CONTENT, CaptureType.REQUEST_HEADERS);

        open("http://testerhome.com/report/");

    }

    @AfterMethod
    public void afterMethod() {
        WebDriverRunner.closeWebDriver(); //注意，防止代理影响其他用例，结束后关闭 Driver

    }

    @Test
    public void downloadReport() {

        $("#report-type").selectOptionContainingText("日报");
        $("#report-date").setValue("2016-08-03");

        proxy.newHar("report");

        $("#report-download").click();
        sleep(1000);

        StringWriter writer = new StringWriter();
        try {
            proxy.getHar().writeTo(writer);
        } catch (IOException e) {
            e.printStackTrace();
        }
        String harAsString = writer.toString();
        Object document = com.jayway.jsonpath.Configuration.defaultConfiguration().jsonProvider().parse(harAsString);

        JSONArray reqUrl = JsonPath.read(document, "$..url");
        assertThat(reqUrl.get(0)).isEqualTo("http://testerhome.com/report?type=day&date=2016-08-03");

    }
}
```

埋点：  

```java
public class Recommend {
    BrowserMobProxy proxy;

    @BeforeMethod
    public void beforeMethod() {
        com.codeborne.selenide.Configuration.browser="chrome";
        proxy = new BrowserMobProxyServer();
        proxy.start(0);
        Proxy seleniumProxy = ClientUtil.createSeleniumProxy(proxy);
        WebDriverRunner.closeWebDriver(); //注意，需要在开启 Driver 前设置代理，需要根据自己用例组织情况决定是否重启 Driver
        WebDriverRunner.setProxy(seleniumProxy);
        proxy.enableHarCaptureTypes(CaptureType.RESPONSE_HEADERS, CaptureType.REQUEST_CONTENT, CaptureType.RESPONSE_CONTENT, CaptureType.REQUEST_HEADERS);

    }

    @AfterMethod
    public void afterMethod() {
        WebDriverRunner.closeWebDriver(); //注意，防止代理影响其他用例，结束后关闭 Driver

    }

    @Test
    public void recommendStatistics() {
        String  recommend = "{\n" +
                "    \"content-id\": 508,\n" +
                "    \"content\": \"TestHome\",\n" +
                "    \"url\": \"http://testerhome.com/\"\n" +
                "}";

        // mock 推荐位内容
        proxy.addResponseFilter(new ResponseFilter() {
            @Override
            public void filterResponse(HttpResponse response, HttpMessageContents contents, HttpMessageInfo messageInfo) {
                String url = messageInfo.getUrl();
                if (url.contains("/recommend")) {
                    contents.setTextContents(recommend);
                }
            }
        });

        proxy.newHar("report");

        open("http://testerhome.com/");
        $("#recommend").scrollTo().shouldBe(Condition.visible);
        assertShow(Integer.valueOf(508));

    }

    private void assertShow(Integer expectContentId) {
        StringWriter writer = new StringWriter();
        try {
            Thread.sleep(2000);
            proxy.getHar().writeTo(writer);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        String harAsString = writer.toString();

        Object docReqBodys = com.jayway.jsonpath.Configuration.defaultConfiguration().jsonProvider().parse(harAsString);
        List<String> reqBodys = JsonPath.read(docReqBodys, "$..[?(@.url=='http://testerhome.com/statistics' && @.method=='POST')]..text");

        // 一个页面一般有多个位置埋点，异步请求发出的顺序也也是不定的
        HashMap idAndEvent = new HashMap();
        reqBodys.forEach((reqBody) -> {
            Object docReqBody = com.jayway.jsonpath.Configuration.defaultConfiguration().jsonProvider().parse(reqBody);
            List<Object> actualContentIds = JsonPath.read(docReqBody, "$..content-id");
            List<String> actualEvents = JsonPath.read(docReqBody, "$..event");

            idAndEvent.put(actualContentIds.get(0), actualEvents.get(0));

        });

        assertThat(idAndEvent).containsKey(expectContentId);
        assertThat(idAndEvent.get(expectContentId)).isEqualTo("show");

    }

}
```