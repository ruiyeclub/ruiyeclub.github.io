---
title: SpringBoot实现图片上传demo以及Nginx进行代理显示
date: 2022-02-12 17:28:11
tags:
---

公司项目需要一个图片上传的功能，就图片能上传到服务器（公司用的windows服务器），然后nginx能进行代理访问到就行了，先简单介绍一下nginx，然后再来实现功能。

# 一、nginx简介

> Nginx (engine x) 是一个高性能的HTTP和反向代理web服务器，特点是占有内存少，并发能力强，事实上nginx的并发能力在同类型的网页服务器中表现较好。
Nginx专门为性能优化而开发，性能是其最重要的考量，实际上非常注重效率，能经起高负载的考验，有报告表明能支持高达50000个并发连接数。

# 二、反向代理

1.正向代理
在客户端（浏览器）配置代理服务器，通过代理服务器进行互联网访问。
2.反向代理
我们只需要将请求发送到反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，再返回给客户端，此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器IP地址。

# 三、负载均衡

增加服务器的数量，然后将请求分发到各个服务器上，将原先请求集中到单个服务器上的情况改为分发到多个服务器上，将负载分发到不同的服务器，也就是我们所说的负载均衡。

# 四、动静分离


为了加快网站的解析速度，可以把动态页面和静态页面由不同的服务器来解析，加快解析速度。减低原来单个服务器的压力。

# 五、nginx常用命令

1.使用nginx操作命令前提条件：必须进行nginx的目录

2.查看nginx的版本号
nginx -v

3.启动nginx
nginx

4.关闭nginx
nginx -s stop

5.重新加载nginx
nginx -s reload

# 六、nginx的配置文件（nginx.conf）

nginx配置文件有三部分组成
1.全局块
从配置文件开始到events块之间的内容，主要会设置一些影响nginx服务器整体运行的配置指令。
比如：worker_processes 1; worker_processes值越大，可以支持的并发处理量也越多。

2.events块

events块涉及的指令主要影响nginx服务器与用户的网络连接。
比如：worker_connection 1024; 支持的最大连接数。

3.http块
nginx服务器配置中最频繁的部分，http块也可以包括http全局块、server块。

# 七、nginx配置图片的访问路径

图片文件上传至服务器D:/images中，然后通过IP地址/upload/加图片名称进行访问。

```xml
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        #访问路径拼接 upload 访问本地绝对路径下的某图片 
        location /upload/ {
        alias D:/images/;
        autoindex on;
        }

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```
配置好nginx.conf记得重启一下服务器。效果如图：
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/83f770e8dad9c1ca09033b2e9ce6037f.png)

# 八、java后台代码

```java
/**
 * 文件上传
 */
@RestController
public class FileController {

    @PostMapping(value = "/fileUpload")
    public String fileUpload(@RequestParam(value = "file") MultipartFile file) {
        if (file.isEmpty()) {
            System.out.println("请选择图片");
        }
        String fileName = file.getOriginalFilename();  // 文件名
        String suffixName = fileName.substring(fileName.lastIndexOf("."));  // 后缀名
        String filePath = "D:/images/"; // 上传后的路径
        fileName = UUID.randomUUID() + suffixName; // 新文件名
        File dest = new File(filePath + fileName);
        if (!dest.getParentFile().exists()) {
            dest.getParentFile().mkdirs();
        }
        try {
            file.transferTo(dest);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //返回图片名称
        return fileName;
    }
}
```
这里将图片上传至D:/images文件夹中，因为前面配置了nginx的缘故，我这里是在本地测试的，直接使用localhost/upload/+返回的图片名称就可以访问到了。

如果想要通过项目的地址外加端口号进行访问的话，可以配置一个资源映射路径。

```java
/**
 * 资源映射路径
 */
@Configuration
public class MyWebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/upload/**").addResourceLocations("file:D:/images/");
    }
}
```
like this!
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/ef6fa062cb4f717a5d1287f49357592f.png)

项目代码可在[github](github)进行查看
