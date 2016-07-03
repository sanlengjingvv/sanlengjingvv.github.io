---
layout: post
title: Windows 环境变量、用户、 Jenkins 工作路径
date: 2015-08-13 09:46
disqus: y
---

#### 一、command 路径

在 C 盘新建文件夹 command ，在 command 中新建文件 demonstration.bat ，文件中添加`echo This is demonstration.bat`。
打开 cmd ，执行下述命令并查看结果：  
  
```PowerShell
C:\Users\Administrator>cd C:\command

C:\command>demonstration.bat
This is demonstration.bat

C:\command>cd ..

C:\>demonstration.bat
'demonstration.bat' 不是内部或外部命令，也不是可运行的程序
或批处理文件。

C:\>C:\command\demonstration.bat
This is demonstration.bat
```
  
得到的经验：
1、命令不加路径时在当前路径搜索可执行文件。
2、命令加路径时当前路径无影响。

#### 二、环境变量，重启

新建 **用户变量**，变量名 DemonstrationPath ，变量值 C:\command 。
执行：
  
```PowerShell
C:\>echo %DemonstrationPath%
%DemonstrationPath%

C:\>%DemonstrationPath%\demonstration.bat
系统找不到指定路径
```
  
重启 cmd ，执行：
  
```PowerShell
C:\Users\Administrator>echo %DemonstrationPath%
C:\command

C:\Users\Administrator>%DemonstrationPath%\demonstration.bat
This is demonstration.bat
```
  
得到的经验：
打开 cmd 后修改环境变量，在已打开的 cmd 中环境变量不生效。

#### 三、环境变量，用户变量和系统变量

新建一个 windows 用户，用户名 testerhome ，切换用户到 testerhome 。
执行：
  
```PowerShell
C:\Users\testerhome>whoami
win-716fc3ul8ml\testerhome

C:\Users\testerhome>echo %DemonstrationPath%
%DemonstrationPath%

C:\Users\testerhome>%DemonstrationPath%\demonstration.bat
系统找不到指定的路径。
```
  
切换到 Administrator 用户。
删除 **用户变量** DemonstrationPath ，新建 **系统变量**，变量名 DemonstrationPath ，变量值 C:\command ，重启 cmd 。
执行：
  
```PowerShell
C:\Users\Administrator>whoami
win-716fc3ul8ml\administrator

C:\Users\Administrator>echo %DemonstrationPath%
C:\command

C:\Users\Administrator>%DemonstrationPath%\demonstration.bat
This is demonstration.bat
```
  
切换到 testerhome 用户，重启 cmd 。
执行：
  
```PowerShell
C:\Users\testerhome>whoami
win-716fc3ul8ml\testerhome

C:\Users\testerhome>echo %DemonstrationPath%
C:\command

C:\Users\testerhome>%DemonstrationPath%\demonstration.bat
This is demonstration.bat
```
  
切换到 Administrator 用户，新建 **用户变量**，变量名 DemonstrationPath ，变量值 C: ，重启 cmd 。
执行：
  
```PowerShell
C:\Users\Administrator>echo %DemonstrationPath%
C:

C:\Users\Administrator>%DemonstrationPath%\demonstration.bat
'C:\demonstration.bat' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```
  
得到的经验：
1、不同 Windows 用户使用自己的 **用户变量**。
2、不同 Windows 用户使用同样的 **系统变量**。
3、有同名的 **系统变量**和 **用户变量**时，优先使用 **用户变量**。

#### 四、系统变量 Path
在 cmd 执行：
  
```PowerShell
C:\Users\Administrator>demonstration.bat
'demonstration.bat' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```
  
查看 **系统变量** Path ，在已有变量后追加 ;%DemonstrationPath% ，重启 cmd 。
执行：
  
```PowerShell
C:\Users\Administrator>demonstration.bat
This is demonstration.bat

C:\Users\Administrator>D:

D:\>demonstration.bat
This is demonstration.bat
```
  
得到的经验：
命令不加路径时在当前路径和 ** Path 变量值中的路径中**搜索可执行文件。

#### 五、Jenkins 工作路径和所属 Windows 用户
下载 [Jenkins.war](http://jenkins-ci.org) 后执行`java -jar jenkins.war`启动 Jenkins 服务。
启动成功后增加一个 Windows 系统变量，变量名 Restart ，变量值 ThisIsRestart 。 
在 Jenkins 中新建自由风格的软件项目，Item 名称是 survey ，增加构建步骤 Execute Windows batch command ，填写：
  
```PowerShell
echo %Restart%
```
  
保存后立即构建，查看 Console Output ：
  
```PowerShell
C:\Users\Administrator\.jenkins\jobs\survey\workspace>echo
ECHO 处于打开状态。
```
  
重启 Jenkins 服务，立即构建 survey ，查看 Console Output ：
  
```PowerShell
C:\Users\Administrator\.jenkins\jobs\survey\workspace>echo ThisIsRestart
ThisIsRestart
```
  
修改 survey 构建命令：
  
```PowerShell
whoami
```
  
保存后立即构建，查看 Console Output ，有部分内容是：
  
```PowerShell
C:\Users\Administrator\.jenkins\jobs\survey\workspace>whoami
heishui-pc\Administrator
```
  
关闭开启 Jenkins 服务的 cmd 窗口，切换 Windows 用户到 testerhome ，执行`java -jar jenkins.war`启动 Jenkins 服务。
新建自由风格的软件项目，Item 名称是 survey ，增加构建步骤 Execute Windows batch command ，填写：
  
```PowerShell
whoami
```
  
保存后立即构建，查看 Console Output ，有部分内容是：
  
```PowerShell
C:\Users\testerhome\.jenkins\jobs\survey\workspace>whoami
heishui-pc\testerhome
```
  
配置 survey ，高级项目选项-使用自定义的工作空间，目录输入 D:\workspace ，保存后立即构建 survey ，查看  Console Output 。
  
```PowerShell
D:\workspace>whoami
heishui-pc\testerhome
```
  
得到的经验：
1、启动 Jenkins 后修改的环境变量，重启 Jenkins 后生效。
2、Jenkins 的默认工作路径是： *Jenkins系统管理-系统设置-主目录 + .jenkins\jobs + Item名称 + workspace* 。
3、可以给不同项目配置各自的工作路径。


#### 六、Jenkins 系统设置中的 JDK 和 Maven
安装 JDK7 和 JDK8 ，Windows环境变量配置到 JDK7 。
解压 Maven3.3.3 和 Maven3.2.5 ，Windows环境变量配置到 Maven3.3.3 。
取消勾选 survey 的使用自定义的工作空间，修改构建命令为：
  
```PowerShell
echo %JAVA_HOME%
java -version
echo %MAVEN_HOME%
mvn -v
```
  
保存后立即构建，查看 Console Output ：
  
```PowerShell
C:\Users\testerhome\.jenkins\jobs\survey\workspace>echo C:\Program Files (x86)\Java\jdk1.7.0_79 
C:\Program Files (x86)\Java\jdk1.7.0_79

C:\Users\testerhome\.jenkins\jobs\survey\workspace>java -version 
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) Client VM (build 24.79-b02, mixed mode, sharing)

C:\Users\testerhome\.jenkins\jobs\survey\workspace>echo D:\Program Files\apache-maven-3.3.3\bin 
D:\Program Files\apache-maven-3.3.3\bin

C:\Users\testerhome\.jenkins\jobs\survey\workspace>mvn -v 
Apache Maven 3.3.3 (7994120775791599e205a5524ec3e0dfe41d4a06; 2015-04-22T19:57:37+08:00)
Maven home: D:\Program Files\apache-maven-3.3.3\bin\..
Java version: 1.7.0_79, vendor: Oracle Corporation
Java home: C:\Program Files (x86)\Java\jdk1.7.0_79\jre
Default locale: zh_CN, platform encoding: GBK
OS name: "windows 7", version: "6.1", arch: "x86", family: "windows"
```
  
在 Jenkins 系统管理-系统设置中，新增 JDK ，不勾选自动安装，在 JAVA_HOME 填写 JDK8 的路径。 
新增 Maven ，不勾选自动安装，在 MAVEN_HOME 填写 maven3.2.5 的路径。 
保存后立即构建 survey ，查看 Console Output ：
  
```PowerShell
C:\Users\testerhome\.jenkins\jobs\survey\workspace>echo C:\Program Files\Java\jdk1.8.0_51 
C:\Program Files\Java\jdk1.8.0_51

C:\Users\testerhome\.jenkins\jobs\survey\workspace>java -version 
java version "1.8.0_51"
Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)

C:\Users\testerhome\.jenkins\jobs\survey\workspace>echo D:\Program Files\apache-maven-3.3.3\bin 
D:\Program Files\apache-maven-3.3.3\bin

C:\Users\testerhome\.jenkins\jobs\survey\workspace>mvn -v 
Apache Maven 3.3.3 (7994120775791599e205a5524ec3e0dfe41d4a06; 2015-04-22T19:57:37+08:00)
Maven home: D:\Program Files\apache-maven-3.3.3\bin\..
Java version: 1.8.0_51, vendor: Oracle Corporation
Java home: C:\Program Files\Java\jdk1.8.0_51\jre
Default locale: zh_CN, platform encoding: GBK
OS name: "windows 7", version: "6.1", arch: "amd64", family: "dos"
```
  
新建项目，构建一个 maven 项目，Item 名称 MavenSurvey ，增加 Pre Steps - Execute Windows batch command ，填写：
  
```PowerShell
echo %JAVA_HOME%
java -version
echo %MAVEN_HOME%
mvn -v
```
  
保存后立即构建 MavenSurvey ，查看 Console Output ：
  
```PowerShell
C:\Users\testerhome\.jenkins\jobs\MavenSurvey\workspace>echo C:\Program Files\Java\jdk1.8.0_51 
C:\Program Files\Java\jdk1.8.0_51

C:\Users\testerhome\.jenkins\jobs\MavenSurvey\workspace>java -version 
java version "1.8.0_51"
Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)

C:\Users\testerhome\.jenkins\jobs\MavenSurvey\workspace>echo D:\Program Files\apache-maven-3.2.5 
D:\Program Files\apache-maven-3.2.5

C:\Users\testerhome\.jenkins\jobs\MavenSurvey\workspace>mvn -v 
Apache Maven 3.2.5 (12a6b3acb947671f09b81f49094c53f426d8cea1; 2014-12-15T01:29:23+08:00)
Maven home: D:\Program Files\apache-maven-3.2.5
Java version: 1.8.0_51, vendor: Oracle Corporation
Java home: C:\Program Files\Java\jdk1.8.0_51\jre
Default locale: zh_CN, platform encoding: GBK
OS name: "windows 7", version: "6.1", arch: "amd64", family: "dos"
```
  
得到的经验：
1、自由风格的项目，JDK 优先使用 Jenkins 系统设置中的路径，Maven 使用 Windows 环境变量中的路径。
2、Maven 项目，JDK 优先使用 Jenkins 系统设置中的路径，Maven 优先使用 Jenkins 系统设置中的路径。
