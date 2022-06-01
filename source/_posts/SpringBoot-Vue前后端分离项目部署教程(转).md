---
title: SpringBoot+Vue前后端分离项目部署教程(转)
date: 2022-06-01 09:39:56
tags:
---

## 1.打包后端项目jar包
打开pom.xml文件，修改packaging方式为jar
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/1b2cb327efd7ff924aed08f1b823c4a4.png)

点击右侧maven插件 -> package
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/13824b79027f796b472c606e9fd34358.png)

打包成功后会在target目录下生成jar包
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/56a039e0a216ddb1119aea33946b87e0.png)

## 2.编写Dockerfile文件
```sh
FROM java:8
VOLUME /tmp
ADD blog-springboot-0.0.1.jar blog.jar       
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/blog.jar"] 
sh
FROM java:8
VOLUME /tmp
ADD blog-springboot-0.0.1.jar blog.jar       
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/blog.jar"] 
```
ps：Dockerfile文件不需要后缀，直接为文件格式

## 3.编写blog-start.sh脚本
```sh
#源jar路径  
SOURCE_PATH=/usr/local/docker
#docker 镜像/容器名字或者jar名字 这里都命名为这个
SERVER_NAME=blog-springboot-0.0.1.jar
TAG=latest
SERVER_PORT=8080
#容器id
CID=$(docker ps | grep "$SERVER_NAME" | awk '{print $1}')
#镜像id
IID=$(docker images | grep "$SERVER_NAME:$TAG" | awk '{print $3}')
if [ -n "$CID" ]; then
  echo "存在容器$SERVER_NAME, CID-$CID"
  docker stop $SERVER_NAME
  docker rm $SERVER_NAME
fi
# 构建docker镜像
if [ -n "$IID" ]; then
  echo "存在$SERVER_NAME:$TAG镜像，IID=$IID"
  docker rmi $SERVER_NAME:$TAG
else
  echo "不存在$SERVER_NAME:$TAG镜像，开始构建镜像"
  cd $SOURCE_PATH
  docker build -t $SERVER_NAME:$TAG .
fi
# 运行docker容器
docker run --name $SERVER_NAME -v /usr/local/upload:/usr/local/upload -d -p $SERVER_PORT:$SERVER_PORT $SERVER_NAME:$TAG
echo "$SERVER_NAME容器创建完成"
sh
#源jar路径  
SOURCE_PATH=/usr/local/docker
#docker 镜像/容器名字或者jar名字 这里都命名为这个
SERVER_NAME=blog-springboot-0.0.1.jar
TAG=latest
SERVER_PORT=8080
#容器id
CID=$(docker ps | grep "$SERVER_NAME" | awk '{print $1}')
#镜像id
IID=$(docker images | grep "$SERVER_NAME:$TAG" | awk '{print $3}')
if [ -n "$CID" ]; then
  echo "存在容器$SERVER_NAME, CID-$CID"
  docker stop $SERVER_NAME
  docker rm $SERVER_NAME
fi
# 构建docker镜像
if [ -n "$IID" ]; then
  echo "存在$SERVER_NAME:$TAG镜像，IID=$IID"
  docker rmi $SERVER_NAME:$TAG
else
  echo "不存在$SERVER_NAME:$TAG镜像，开始构建镜像"
  cd $SOURCE_PATH
  docker build -t $SERVER_NAME:$TAG .
fi
# 运行docker容器
docker run --name $SERVER_NAME -v /usr/local/upload:/usr/local/upload -d -p $SERVER_PORT:$SERVER_PORT $SERVER_NAME:$TAG
echo "$SERVER_NAME容器创建完成"
```
ps：sh文件需要用notepad++转为Unix格式
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/aaed807d8df80aae11a94ea00df2264c.png)

## 4.将文件传输到服务器
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/183030180db2eb36f8949a2c81cb02ea.png)

将上述三个文件传输到/usr/local/docker下（手动创建文件夹）
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/d84ab03c0e7b56ee01dce5f89d30d554.png)

## 5.docker运行后端项目
进入服务器/usr/local/docker下，构建后端镜像
```sh
sh ./blog-start.sh 
```

![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/9a59bf2c7c8c465b707e5645a4b175b1.png)
ps：第一次时间可能比较长，耐心等待即可

查看是否构建成功
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/1096233de4147cd81b65728a495914a7.png)

可以去测试下接口是否运行成功
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/a91b807200f73c0c406eae9ffb1e4297.png)
ps：需要重新部署只需重新传jar包，执行sh脚本即可

## 6.打包前端项目
打开cmd，进入Vue项目路径 -> npm run build
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/718358e5cc8e32b6ed1b0e0788dd7e22.png)

打包成功后会在目录下生成dist文件
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/0b0b7a9a815eeef02e658c15ea0b564b.png)

将Vue打包项目传输到/usr/local/vue下（由于我前台和后台分为两个项目，所以改名dist文件）
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/d8eb85b7f7bd34b93b1d2ac9b24142fb.png)

## 7.nginx配置(有域名选这个)
在/usr/local/nginx下创建nginx.conf文件，格式如下
```sh
events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    client_max_body_size     50m;
    client_body_buffer_size  10m; 
    client_header_timeout    1m;
    client_body_timeout      1m;

    gzip on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_comp_level  4;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;

server {
        listen       80;
        server_name  前台域名;
     
        location / {		
            root   /usr/local/vue/blog;
            index  index.html index.htm; 
            try_files $uri $uri/ /index.html;	
        }
			
	location ^~ /api/ {		
            proxy_pass http://你的ip:8080/;
	    proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;						
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
		
    }
	
server {
        listen       80;
        server_name  后台子域名;
     
        location / {		
            root   /usr/local/vue/admin;
            index  index.html index.htm; 
            try_files $uri $uri/ /index.html;	
        }
			
	location ^~ /api/ {		
            proxy_pass http://你的ip:8080/;
	    proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;						
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
		
    }

server {
        listen       80;
        server_name  websocket子域名;
     
        location / {
          proxy_pass http://你的ip:8080/websocket;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_set_header Host $host:$server_port;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
       }
	
    }

server {
        listen       80;
        server_name  上传文件子域名;
     
        location / {		
          alias /usr/local/upload/; 
          autoindex on;
          autoindex_exact_size on;
          autoindex_localtime on;
        }		
		
    }
 
 }
```
ps：我前台和后台时分为两个域名，所以写了两个server，前端项目路径为之前传输的路径，其他两个为文件上传域名和websocket转发域名。

docker启动nginx服务
```sh
docker run --name nginx --restart=always -p 80:80 -d -v /usr/local/nginx/nginx.conf:/etc/nginx/nginx.conf -v /usr/local/vue:/usr/local/vue -v /usr/local/upload:/usr/local/upload nginx 
```

## 8.nginx配置(无域名选这个)
```sh
events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    client_max_body_size     50m;
    client_body_buffer_size  10m; 
    client_header_timeout    1m;
    client_body_timeout      1m;

    gzip on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_comp_level  4;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;

server {
        listen       80;
        server_name  你的ip;
     
        location / {		
            root   /usr/local/vue/blog;
            index  index.html index.htm; 
            try_files $uri $uri/ /index.html;	
        }
			
	location ^~ /api/ {		
            proxy_pass http://你的ip:8080/;
	    proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;						
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
		
    }
	
server {
        listen       81;
        server_name  你的ip;
     
        location / {		
            root   /usr/local/vue/admin;
            index  index.html index.htm; 
            try_files $uri $uri/ /index.html;	
        }
			
	location ^~ /api/ {		
            proxy_pass http://你的ip:8080/;
	    proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;						
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
		
    }

server {
        listen       82;
        server_name  你的ip;
     
        location / {
          proxy_pass http://你的ip:8080/websocket;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_set_header Host $host:$server_port;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
       }
	
    }

server {
        listen       83;
        server_name  你的ip;
     
        location / {		
          alias /usr/local/upload/; 
          autoindex on;
          autoindex_exact_size on;
          autoindex_localtime on;
        }		
		
    }
 
 }
```
docker启动nginx服务
```sh
docker run --name nginx --restart=always -p 80:80 -p 81:81 -p 82:82 -p 83:83 -d -v /usr/local/nginx/nginx.conf:/etc/nginx/nginx.conf -v /usr/local/vue:/usr/local/vue -v /usr/local/upload:/usr/local/upload nginx 
```
ps：需要通过ip + 端口号访问项目

## 9.运行测试
去浏览器测试下是否运行成功
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/66732d08619276eea4be2f8ab7f32551.png)

## 10.其他设置
进入后台管理 -> 网站管理 -> 其他设置，配置websocket域名
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/a97b2594f13b2928072728067ac68968.png)

## 11.总结
整个前后端分离的部署教程到这里就结束啦，第一次部署可能会比较麻烦，不过后面就轻车熟路了，有问题的可以私聊问我或者在评论区留言。