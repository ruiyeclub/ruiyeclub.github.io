---
title: Docker快速上手之部署SpringBoot项目
date: 2022-02-01 17:35:40
tags:
---

**Docker是基于Go语言实现的云开源项目。**

Docker的主要目标是“Build，Ship and Run Any App,Anywhere”，也就是通过对应用组件的封装、分发、部署、运行等生命周期的管理，使用户的APP（可以是一个WEB应用或数据库应用等等）及其运行环境能够做到“一次封装，到处运行”。
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/2db39f1e0972b7dd1557c891d9300fdd.png)
Linux 容器技术的出现就解决了这样一个问题，而 Docker 就是在它的基础上发展过来的。将应用运行在 Docker 容器上面，而 Docker 容器在任何操作系统上都是一致的，这就实现了跨平台、跨服务器。只需要一次配置好环境，换到别的机子上就可以一键部署好，大大简化了操作。

####  一、我所理解的Docker
> 我喜欢将Docker比喻成"方便面"，为什么说是方便面，之前部署项目环境的配置十分麻烦，换一台机器，就要重来一次，费力费时。举个栗子，我们部署一个SpringBoot项目，我们需要在服务器上面配置项目的运行环境，要安装各种各样的软件JDK/MySQL/Redis/nginx，安装和配置这些东西十分麻烦，下次需要换个服务器重新部署又得重新安装一遍，简直要命。而Docker能将项目连带着运行环境一同部署过去，就好像泡面，泡面所需的调料包还有工具都附加在了里面。一句话总结Docker的好处，Docker解决了运行环境和配置问题，方便做持续集成并有助于整体发布的容器虚拟化技术。

#### 二、Docker与虚拟机的区别

||Docker容器|虚拟机（VM）|
|-|-|-|
|操作系统|与宿主机共享OS|宿主机OS上运行虚拟机OS|
|存储大小|镜像小，便于存储与传输|镜像庞大（vmdk、vdi等）|
|运行性能|几乎无额外性能损失|操作系统额外的CPU、内存消耗|
|移植性|轻便、灵活，适用于Linux|笨重，与虚拟化技术耦合度高|
|硬件亲和性|面向软件开发者|面向硬件运维者|

#### 三、Docker里面三个重要的概念Dockerfile、镜像(image)、容器(Container)
1.Dockerfile是用来构建Docker镜像的构建文件，是由一系列命令和参数构成的脚本。

Dockerfile体系结构：
```xml
FROM　　基础镜像，当前新镜像是基于哪个镜像的
MAINTAINER　　镜像维护者的姓名和邮箱地址
RUN　　容器构建时需要运行的命令
EXPOSE　　当前容器对外暴露出的端口
WORKDIR　　指定在创建容器后，终端默认登录进来的工作目录，一个落脚点
ENV　　用来在构建镜像过程中设置环境变量
AD　　将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
COPY　　类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中<源路径>的文件/目录复制到新的一层的镜像内的<目标路径>位置
VOLUME　　容器数据卷，用于数据保存和持久化工作
CMD　　指定一个容器启动时要运行的命令。Dockerfile中可以有多个CMD指令，但只要最后一个生效，CMD会被docker run之后的参数替换
ENTRYPOINT　　指定一个容器启动时要运行的命令。ENTRYPOINT的目的和CMD一样，都是在指定容器启动程序及参数
ONBUILD　　当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发
```
2.镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。

3.容器是用镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。可以把容器看做是一个简易版的Linux环境（包括root用户权限、用户空间和网络空间等）和运行在其中的应用程序。

#### 四、安装Docker

1.安装Docker依赖包

```xml
yum install -y yum-utils device-mapper-persistent-data lvm2
```

2.设置阿里云镜像源
```xml
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```xml
3.安装 Docker-CE
```xml
yum install docker-ce
```
4.查看是否安装成功
```xml
docker version
```
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/3d0de6d7f169b639389384e8184829f5.png)

#### 五、使用Docker安装MySQL

1.使用Docker拉取Mysql镜像(这里安装的是MySQL5.6版本)
```xml
docker pull mysql:5.6
```
2.安装mysql命令
```xml
docker run -d -p 3306:3306 --name mysql -v /ray/mysql/conf:/etc/mysql/conf.d -v /ray/mysql/logs:/logs -v /ray/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=admin -d mysql:5.7
```
命令说明：

```java
-p 3306:3306　　将主机的3306端口映射到docker容器的3306端口
--name mysql　　运行服务器名字
-v /ray/mysql/conf:/etc/mysql/conf.d　　将主机/ray/mysql目录下的conf/my.cnf挂载到容器的/etc/mysql/conf.d
-v /ray/mysql/logs:/logs　　将主机/ray/mysql目录下的logs目录挂载到容器的/logs
-v /ray/mysql/data:/var/lib/mysql　　将主机/ray/mysql目录下的data目录挂载到容器的/var/lib/mysql
-e MYSQL_ROOT_PASSWORD=admin　　初始化root用户的密码
-d mysql:5.7　　后台程序运行mysql5.7
```
3.进入mysql
```xml
docker exec -it MySQL运行成功后的容器ID /bin/bash
```
然后即可登录MySQL，创建数据库，或者使用Navicat等工具创建数据库。

#### 六、使用Docker安装Redis
1.使用Docker拉取Redis镜像(这里安装的是Redis3.2版本)
```xml
docker pull redis:3.2
```
2.安装redis命令(这里不对命令做解释)
```xml
docker run -p 6379:6379 -v /ray/redis/data:/data -v /ray/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf -d redis:3.2 redis-server /usr/local/etc/redis/redis.conf --appendonly yes
```
#### 七、Docker部署SpringBoot项目

1.将项目打包成jar包(假设名字为myblog.jar)，并编写一个Dockerfile文件，文件内容如下：

```xml
# Docker image for springboot file run
# VERSION 0.0.1
# Author: Ray
# 基础镜像使用java
FROM java:8
# 作者
MAINTAINER Ray <185048761@qq.com>
# VOLUME 指定了临时文件目录为/tmp。
# 其效果是在主机 /var/lib/docker 目录下创建了一个临时文件，并链接到容器的/tmp
VOLUME /tmp
# 将jar包添加到容器中并更名为app.jar
ADD myblog.jar app.jar
# 运行jar包
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
#暴露8080端口
EXPOSE 80
```
2.将这两个文件一并上传至服务器中的同一个目录下面，进入该文件夹后执行此命令构建镜像：
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/f175e9402c4d4bf3e98c78f055b6e9c6.png)
```xml
docker build -t myblog.jar .
```
3.生成docker容器，并运行
```xml
docker run -d -p 80:80 myblog.jar
```
等一会儿，SpringBoot项目跑起来了后，就可以使用浏览器通过80端口进行访问了