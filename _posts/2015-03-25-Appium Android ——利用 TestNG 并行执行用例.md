---
layout: post
title: Appium Android ——利用 TestNG 并行执行用例
date: 2015-03-25 20:02
disqus: y
---

一、测试类*注1
```java
package com.testerhome;

import io.appium.java_client.android.AndroidDriver;

import java.net.MalformedURLException;
import java.net.URL;

import org.openqa.selenium.By;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.testng.annotations.BeforeSuite;
import org.testng.annotations.Parameters;
import org.testng.annotations.Test;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.AfterClass;

public class Suite1 {
	public String port;
	public String udid;
	private AndroidDriver driver;
	
  @Test
  public void switches() throws InterruptedException {
	  WebElement sound = driver.findElementByAndroidUIAutomator("new UiSelector().text(\"Sound\")");
	  sound.click();
	  System.out.println("checked");
	  Thread.sleep(2000);
	  System.out.println(Thread.currentThread());
  }
  
  @BeforeSuite
  @Parameters({ "port", "udid" })
  public void beforeSuite(String port, String udid) {
	  this.port = port;
	  this.udid = udid;
	  
  }
  
  @BeforeClass
  public void beforeClass() throws MalformedURLException{
		System.out.println(port + udid);
		
	  	DesiredCapabilities capabilities = new DesiredCapabilities();
		capabilities.setCapability("deviceName","device");
	  	capabilities.setCapability("automationName","Appium");
	  	capabilities.setCapability("platformVersion", "4.4");
	  	capabilities.setCapability("udid", udid);
	  	capabilities.setCapability("appPackage", "com.android.settings");
	  	capabilities.setCapability("appActivity", ".Settings");
		driver = new AndroidDriver(new URL("http://127.0.0.1:" + port + "/wd/hub"), capabilities);
		
  }

  @AfterClass
  public void afterClass() {
	  driver.quit();
	  
  }

}
```

二、连接两个Android设备或启动两个虚拟机
使用
`adb devices`
获取udid

三、项目路径下新建两个testng.xml
testng1.xml
```
<?xml version="1.0" encoding="UTF-8"?>  
<suite name="Suite1">
  <parameter name = "port" value = "4723"/>
  <parameter name = "udid" value = "emulator-5554"/>
  <test name="Test">  
    <classes>  
      <class name="com.testerhome.Suite1"/> 
    </classes>  
  </test>  
</suite>  
```

testng2.xml
```
<?xml version="1.0" encoding="UTF-8"?>  
<suite name="Suite2">
  <parameter name = "port" value = "4725"/>
  <parameter name = "udid" value = "emulator-5556"/>
  <test name="Test">  
    <classes>  
      <class name="com.testerhome.Suite1"/> 
    </classes>  
  </test>  
</suite>  
```

四、开启两个appium server*注2、注3
第一个：
Port:4723
bootstrapPort:4724

第二个：
Port:4725
bootstrapPort:4726

五、导出依赖*注4
因为是用maven工程创建的，所以先导出依赖到项目路径下的lib文件夹
`mvn dependency:copy-dependencies -DoutputDirectory=lib`

六、执行测试
先用Maven串行执行一次以编译出Class文件
`mvn clean test`
然后
`java -classpath ".\target\test-classes" -Djava.ext.dirs=lib org.testng.TestNG -suitethreadpoolsize 2 testng1.xml testng2.xml`
如果没有配置TestNG环境变量
`java -classpath ".\target\test-classes;D:\Programs\testng-6.8\testng-6.8.jar" -Djava.ext.dirs=lib org.testng.TestNG -suitethreadpoolsize 2 testng1.xml testng2.xml`

七、查看报告
默认在项目路径下的test-output文件夹

注1：
这个测试类没有指定app路径，如果指定，同时unzip的时候会冲突。目前是复制了多个apk。
File app = new File(appDir, "AppName"+port+".apk");
并在appium server指定不同的临时文件路径，比如：
--tmp D:\tem1
--tmp D:\tem2

注2：
两个端口的介绍可以看这两个链接：
appium 自动化测试教程 ppt(第二版)
http://testerhome.com/topics/284
Appium Android Bootstrap源码分析之简介
http://blog.csdn.net/zhubaitian/article/details/40619777

注3：
如果使用到Selendroid或Chromium，还需要指定其他端口（需要修改测试类）
Selendroid port:8080
Selendroid port:8081
Chromium port:9515
Chromium port:9516

注4：
本来准备直接用mvn test并行执行的，但没试出来传suitethreadpoolsize参数的办法
