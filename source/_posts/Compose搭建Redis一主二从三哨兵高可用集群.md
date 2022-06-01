---
title: Compose搭建Redis一主二从三哨兵高可用集群
date: 2022-02-10 17:35:40
tags:
---

## 一、Docker Compose介绍
> https://docs.docker.com/compose/

Docker官方的网站是这样介绍Docker Compose的：

**Compose是用于定义和运行多容器Docker应用程序的工具。通过Compose，您可以使用YAML文件来配置应用程序的服务。然后，使用一个命令，就可以从配置中创建并启动所有服务。**

这里Docker Compose给我的感受就是便捷、快速。只需编写一个docker-compose.yml文件，然后通过命令docker-compose up -d，这里就可以搭建多个服务起来，非常适合搭建集群环境。

## 二、安装Docker Compose工具

通过命令安装Compose工具，安装Compose的前提是安装了Docker环境，如何安装和快速使用Docker，可以翻看我之前的那篇博客--[](Docker快速上手之搭建SpringBoot项目)，这里不做赘述了。
```xml
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
```
给安装脚本添加执行权限
```xml
sudo chmod +x /usr/local/bin/docker-compose
```
这里可以使用命令查看是否安装成功
```xml
docker-compose -v
```
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/51ea38aadbb2ff6e09252ef863d8b8ef.png)
到这里Compose工具就安装好了

## 三、使用Compose搭建Redis主从服务器

**主从复制**：主节点负责写数据，从节点负责读数据，主节点定期把数据同步到从节点保证数据的一致性。

从性能方面，redis复制功能增加了读的性能，理论上说，每增加一倍的服务器，整个系统的读能力就增加一倍。

选择好路径，通过命令创建文件夹redis，然后进入文件夹创建docker-compose.yml，或者在本地创建，然后通过Xftp工具上传至服务器。docker-compose.yml写入如下内容。

```xml
version: '2'
services:
  master:
    image: redis　　　　## 镜像
    container_name: redis-master　　　　##容器别名
    command: redis-server --requirepass 123456　　　　##redis密码
    ports:
    - "6379:6379"　　　　##暴露端口
    networks:
    - sentinel-master
  slave1:
    image: redis
    container_name: redis-slave-1
    ports:
    - "6380:6379"
    command: redis-server --slaveof redis-master 6379 --requirepass 123456 --masterauth 123456 
    depends_on:
    - master
    networks:
    - sentinel-master
  slave2:
    image: redis
    container_name: redis-slave-2
    ports:
    - "6381:6379"
    command: redis-server --slaveof redis-master 6379 --requirepass 123456 --masterauth 123456
    depends_on:
    - master
    networks:
    - sentinel-master
```
启动redis集群
```xml
docker-compose up -d
```
docker ps查看运行的实例，这里我们需要用到主节点的ip地址，注意不是电脑的ip哦，是节点的ip地址。
```xml
docker inspect 主节点容器id
```
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/53f49345df59e36ee71202f08c7f8d94.png)
这里先记着这个ip地址，后面使用到哨兵需要绑定这个主节点的ip。

## 四、哨兵sentinel模式搭建

上面我们已经搭建好了主从模式，当主服务器宕机后，需要手动把一台服务器切换成主服务器，这里需要人工干预，费事费力，还会造成一段时间内服务不可用。这不是一种推荐的方式，更多时候，我们优先考虑**哨兵模式**。
哨兵是redis高可用的解决方案，由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器以及这些主服务器属下的所有从服务器，它能够在被监视的主服务器下线时，自动将该主服务器属下的某个优先的从服务器升级为新的主服务器，由这个主服务器代替已下线的主服务器继续处理命令请求。
 
哨兵的功能：
1.监控：哨兵会不断地监控检测主节点和从节点是否运作正常。
2.自动故障转移：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
3.通知：哨兵可以将故障转移的结果发送给客户端。
4.配置提供者：客户端在初始化时，通过连接哨兵来获得当前Redis服务的主节点地址。
 
这里创建sentinel文件夹，然后同样编写docker-compose.yml文件
```xml
version: '2'
services:
  sentinel1:
    image: redis       ## 镜像
    container_name: redis-sentinel-1
    ports:
    - "26379:26379"
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - "./sentinel.conf:/usr/local/etc/redis/sentinel.conf"
  sentinel2:
    image: redis                ## 镜像
    container_name: redis-sentinel-2
    ports:
    - "26380:26379"           
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - "./sentinel2.conf:/usr/local/etc/redis/sentinel.conf"
  sentinel3:
    image: redis                ## 镜像
    container_name: redis-sentinel-3
    ports:
    - "26381:26379"           
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - ./sentinel3.conf:/usr/local/etc/redis/sentinel.conf
networks:
  default:
    external:
      name: redis_sentinel-master
```
继续在此目录编写文件，编写sentinel.conf文件
```xml
port 26379
dir /tmp
#172.18.0.3填写自己的主节点ip
sentinel monitor mymaster 172.18.0.3 6379 2
sentinel auth-pass mymaster 123456 
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000  
sentinel deny-scripts-reconfig yes
```
对命令的解释

> 第三行表示Redis监控一个叫做mymaster的运行在172.18.0.3:6379的master，投票达到2则表示master以及挂掉了。
第四行设置主节点的密码。
第五行表示在一段时间范围内sentinel向master发送的心跳PING没有回复则认为master不可用了。
第六行的parallel-syncs表示设置在故障转移之后，同时可以重新配置使用新master的slave的数量。数字越低，更多的时间将会用故障转移完成，但是如果slaves配置为服务旧数据，你可能不希望所有的slave同时重新同步master。因为主从复制对于slave是非阻塞的，当停止从master加载批量数据时有一个片刻延迟。通过设置选项为1，确信每次只有一个slave是不可到达的。
第七行表示10秒内mymaster还没活过来，则认为master宕机了。

创建好文件后，复制好三份
```xml
cp sentinel.conf sentinel1.conf
cp sentinel.conf sentinel2.conf
cp sentinel.conf sentinel3.conf
```
启动redis哨兵模式
```xml
docker-compose up -d
```
## 五、故障转移测试
上面既然说到了哨兵可以自动转移故障，也就是当主节点宕机的时候，它们能通过选举制度，在从节点中选出一个节点来代替主节点。

我们先来看看主节点此时的信息，这里a288c55db497是我主节点容器的ip，这里先是进入容器，再进入redis客户端，再输入密码，然后使用info查看节点信息。
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/b78632e11091376763eb17eb0a456666.png)
这里可以看到这个节点的身份是master也就是主节点，然后有两个连接着的从节点
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/29b59a2b56a62e32552c35bf626817e4.png)

现在我们停掉这个节点，看看哨兵是否可以实现故障转移
```xml
docker stop a288c55db497
```
然后选择其中一个从节点进入客户端，输入info查看信息
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/81c6dc9c59ee6ff27dd01d9aabad7827.png)
这里发现其中一台从节点已经变成主节点了，而且连接的从节点，也变成了一个。

了解Docker Compose可参考：https://www.jianshu.com/p/658911a8cff3

了解Redis主从哨兵可参考：https://www.cnblogs.com/leeSmall/p/8398401.html