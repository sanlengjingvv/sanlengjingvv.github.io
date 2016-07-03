---
layout: post
title: WebDriver+TestNG+Gradle+Jenkins搭建
date: 2014-08-07 09:29
disqus: y
---

环境及组件：
Windows 7-32   
JDK-7u65  
Gradle-2.0  
Jenkins1.574  
Tomcat-8.0.9  
Eclipse4.2.1  
TestNG6.8.8  
Selenium-java-2.42.2  
 
可能需要代理，可选Smarthosts(注意获取最新host列表)，设置  
203.208.46.200        dl.google.com  
203.208.46.200        dl.l.google.com  
203.208.46.200        dl-ssl.google.com  
 
一、安装JDK  

安装后CMD执行java –version，如成功输出java版本信息。  
 
二、安装Eclipse插件、准备示例项目  

1、安装Gradle插件，eclipse-Help-InstallNew Software... "Work with"编辑框粘贴
http://dist.springsource.com/release/TOOLS/gradle。
勾选Gradle IDE，Next，Next，I Accept，Finish，等待安装完毕，重启Eclipse。
 
2、安装TestNG插件，链接是
http://beust.com/eclipse。安装过程中会出现Security Warning，忽略掉下一步。
 
3、Eclipse-File-New-Other-Gradle-Gradle Project，输入项目名称，Sample project选择JavaQuickstart，Finish。等待……
完成后console会输出：
`Build finished succesfully!`

4、打开build.gradle，修改dependencies{}部分。
```
dependencies {
    compile group: 'commons-collections', name: 'commons-collections', version: '3.2'
    testCompile group: 'org.testng', name: 'testng', version: '6.8.8'
    testCompile group: 'org.seleniumhq.selenium', name: 'selenium-java', version: '2.42.2'
}
```
这里添加了testng和selenium-java依赖，去掉了junit，保存后，右键项目，选Gradle-Refresh Dependencies，等待下载依赖。
 
5、删掉Person.java和PersonTest.java，在src/test/java的org.gradle下新建BaiduSearch类，代码如下(使用了安装在默认路径的Firefox浏览器)。
```java
package org.gradle;
 
importorg.openqa.selenium.WebDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.testng.Assert;
importorg.testng.annotations.Test;
 
public class BaiduSearch{  
    @Test
    public void SearchBaidu(){
        WebDriverdriver = new FirefoxDriver();
        driver.get("http://www.baidu.com");
        Assert.assertTrue(driver.getTitle().contains("百度"));
        driver.quit();
    }
}
```
保存后，Run as TestNG，如果成功console输出：
```
PASSED: SearchBaidu
===============================================
 Default test    Tests run: 1, Failures: 0, Skips: 0
===============================================
```
6、在项目路径下新建testng.xml,内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<suite name="Suite" parallel="false">
  <test name="Test">
    <classes>
      <class name="org.gradle.BaiduSearch"/>
    </classes>
  </test>
</suite>
```

 右键testng.xml，Run As TestNG Suit，如果成功console输出：
```
===============================================
Suite
Total tests run: 1,Failures: 0, Skips: 0
===============================================
```

三、安装Gradle，命令行执行

1、解压gradle-2.0。

2、添加环境变量：
变量名：GRADLE_HOME，变量值：你的Gradle目录。
Path后添加：
`;%GRADLE_HOME%\bin`

3、打开CMD，输入`gradle -version`。可以看到输出Groovy、Ant等版本。
```
Build time:   2014-07-01 07:45:34 UTC
Build number: none
Revision:     b6ead6fa452dfdadec484059191eb641d817226c
                                                      
Groovy:       2.3.3
Ant:          Apache Ant(TM) version 1.9.3 compiled on December 23 2013
JVM:          1.7.0_65 (Oracle Corporation 24.65-b04)
OS:           Windows 7 6.1 x86  
```
4、Eclipse修改build.gradle的test{}部分：
```
test {
    useTestNG(){ 
   options.suites("testng.xml")
    useDefaultListeners = true
         }
}
```
这里设置Gradle使用TestNG的配置进行测试，保存后回到CMD，`cd`到项目路径，执行`gradleclean test`。如成功输出：
`BUILD SUCCESSFUL`  

5、打开 %项目路径%\build\reports\tests\testng-results.xml，可以看到testng的测试报告。
 
四、安装配置Jenkins

 
1、安装tomcat，默认设置。
 
2、将Jenkins.war复制到%Tomcat安装路径%\webapps下，执行%Tomcat安装路径%\bin\Tomcat8.exe（如果使用Monitor的service形式启动，执行过程不会在用户界面显示）。
等待输出
> Run Jenkins isfully up and running  

3、进入http://localhost:8080/jenkins/，系统管理-管理插件-可选插件，勾选testng-plugin，下载待重启后安装。下载完成后重启tomcat。
 
4、进入Jenkins，新建，输入Item名称，选择“构建一个自由风格的软件项目”，OK；
点击高级项目选项栏下的“高级”，勾选“使用自定义的工作空间”，在目录输入Gradle项目路径；
构建栏下“增加构建步骤”，选择Executewindows batch command，命令输入`gradleclean test`；
构建后操作栏下“增加构建后操作步骤”，选择PublishTestNG Results，在TestNG XMLreport pattern中输入“build/reports/tests/testng-results.xm”l（注意是正斜杠，之前TestNG报告的路径）。
保存，进入项目，立即构建。
 
在Jenkins查看Console输出：
```
BUILD SUCCESSFUL
 
Total time: 11.483 secs
 
TestNG Reports Processing: START
Looking for TestNG results report in workspaceusing pattern: build/reports/tests/testng-results.xml
Saving reports...
Processing'C:\Users\Virtual\.jenkins\jobs\tryCI\builds\2014-08-06_20-17-33\testng\testng-results.xml'
TestNG Reports Processing: FINISH
Finished: SUCCESS
```
5、在Jenkins查看TestNG Results，有测试结果。
