# 使用Jenkins简单的配置一个Maven工程

## 配置步骤

### 创建任务

在指定的组里面创建任务，如图所示

![](https://image.ibb.co/b4X3t0/1540179026915.jpg)

### 配置Maven工程

选择maven工程，并且输入一个任务名称

![](https://image.ibb.co/e0Jwff/WX20181022-114048.png)

### 配置git项目地址

选择到源码管理，选择git，分别在下图中不同的框中填写项目git路径，登录的账号名密码，已经分支名称
![](https://image.ibb.co/h6nLqf/WX20181022-130956.png)
这里账号密码是可以选择添加的


### 配置前置处理

![](https://image.ibb.co/hwwAO0/1540185231763.jpg)

```shell
#!/bin/bash 
# set basic env
export DEFAULT_USER_SETTINGS_FILE=/home/satoshi/.m2/settings.xml
cd $WORKSPACE
/opt/apache-maven-3.3.9/bin/mvn clean
```

主要是配置项目启动的宿主机的maven情况。

### 配置Maven打包命令

![](https://image.ibb.co/nDA1wL/WX20181022-131803.png)

```shell
package -DskipTests 
```

一般是 mvn package -DskipTests 但是这里把mvn省了


### 构建后操作

这一步一般就是最终执行启动maven打完的包所使用的shell命令

![](https://image.ibb.co/mqmS30/WX20181022-140606.png)

* 先选择需要启动的服务器：139.219.8.80 
* Source files ： 打包后的jar地址 本项目为 target/eureka-server-demo.jar
* Remove prefix	： target
* Remote directory： 打包后发送到远程服务器的指定文件夹
* Exec command	： 将Maven打包后的文件放在远程服务器指定文件夹后所执行的命令

```shell
kill -9 $((ps -aux | grep eureka | grep -v "grep") |awk '{print $2}' | tail -n +1)
# 根据项目名称关闭项目

sleep 10
# 给定时间停止

nohup java -jar /opt/tomcat/jenkins-demo/eureka-server-demo.jar > /opt/tomcat/jenkins-demo/log.log &
#以后台启动方式 使用java -jar命令启动jar包 

```

最后可以直接访问 [http://139.219.8.80:8761/](http://139.219.8.80:8761/)

