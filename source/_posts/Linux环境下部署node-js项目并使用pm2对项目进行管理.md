---
title: Linux环境下部署node.js项目并使用pm2对项目进行管理
date: 2023-12-21 11:43:52
tags:
---

> node.js是让JavaScript能运行在服务端的开发平台，pm2是一个Node.js 守护进程管理器，可以用于管理和监控Node.js 应用程序。

## 一、安装node环境

### 1.可通过网站下载，进入Node最新版下载 <[https://nodejs.org/en/download/current/](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fnodejs.org%2Fen%2Fdownload%2Fcurrent%2F&source=article&objectId=1772306)

- 通过wget指令下载：`wget https://nodejs.org/dist/v13.11.0/node-v13.11.0-linux-x64.tar.xz`

- 解压：`tar -xvf node-v13.11.0-linux-x64.tar.xz`
- 测试安装是否成功：`cd node-v13.11.0-linux-x64/bin`，执行：`./node -v`



### 2.添加node和npm软链建立链接（**PS:ln命令是一个非常重要命令，它的功能是为某一个文件在另外一个位置建立一个同步的链接**）：

- `ln -s /www/node-v13.11.0-linux-x64/bin/node /usr/local/bin/node`
- `ln -s /www/node-v13.11.0-linux-x64/bin/npm /usr/local/bin/npm`



### 2.1或者是通过改变node包的存放位置进行操作：`mv node-v13.11.0-linux-x64 /usr/local/node`

再编辑配置文件：

```xml
vim /etc/profile

按住i键，进入编辑模式

插入：export PATH=$PATH:/usr/local/node/bin

按esc,输入:wq，退出vim编辑器模式
```



### 3.使用测试命令：`node -v`和`npm -v`



### 4.加速npm

- 使用淘宝的cnpm：`npm install cnpm -g --registry=https://registry.npm.taobao.org`
- 加cnpm软链：`ln -s /www/node-v13.11.0-linux-x64/bin/cnpm /usr/local/bin/cnpm`
- 需要注意，以后使用cnpm去代替npm来执行，比如：`cnpm install XXX`



## 二、安装并使用pm2

> PM2 是一个流行的进程管理器，是在生产环境中后台运行 nodejs 的首选。它提供了很多的功能和选项，包括进程监控、自动重启、负载平衡等等。使用 PM2 后，我们可以方便地将 nodejs 应用程序后台运行。



### 1.安装pm2：`npm install -g pm2`



### 2.添加pm2软链：`ln -s /www/node-v13.11.0-linux-x64/lib/node_modules/pm2/bin/pm2 /usr/local/bin/`



### 3.pm2常用命令：

- 启动指定应用：`pm2 start <script_file|config_file> [options]` ，如：`pm2 start index.js --name httpServer`

- 停止指定应用：`pm2 stop <appName> [options]`，如：`pm2 stop httpServer`

- 查看全部实例：`pm2 list` ，注意：pm2 stop 某个项目后，该项目还会存在pm2 list 的列表里面， 只是状态是 stop, 要想去掉该项目，用pm2 delete

- 重启指定应用：`pm2 reload|restart <appName> [options]`，如：`pm2 restart httpServer`

- 显示指定应用详情：`pm2 show <appName> [options]`，如：`pm2 show httpServer`

- 删除指定应用：`pm2 delete <appName> [options]`，如：`pm2 delete httpServer`，如果修改应用配置行为，最好先删除应用后，重新启动方才生效，如修改脚本入口文件

- 杀掉pm2管理的所有进程`pm2 kill` 

- 查看指定应用的日志，即标准输出和标准错误`pm2 logs <appName>`

- 监控各个应用进程cpu和memory使用情况`pm2 monit`

- 如果项目没有启动就执行 start 如果项目正在运行 就执行relaod`pm2 startOrReload <appName>`