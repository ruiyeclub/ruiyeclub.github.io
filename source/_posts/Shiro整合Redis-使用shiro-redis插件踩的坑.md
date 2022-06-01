---
title: Shiro整合Redis:使用shiro-redis插件踩的坑
date: 2022-06-01 09:36:13
tags:
---

　　一直想在shiro权限这块加入缓存，使用redis是再合适不过了，恰巧已经有大佬将shiro和redis整合在一起使用了，只需在引入pom文件中引入即可。

```xml
<dependency>
      <groupId>org.crazycake</groupId>
      <artifactId>shiro-redis</artifactId>
      <version>3.2.3</version>
</dependency>
```
但是是使用的时候，权限配置这块，也就是重写shiro的doGetAuthorizationInfo方法这里，一直进不来，完整的控制台异常信息如下：

```xml
org.crazycake.shiro.exception.PrincipalInstanceException: class com.company.project.manage.entity.UserInfo must has getter for field: id
We need a field to identify this Cache Object in Redis. So you need to defined an id field which you can get unique id to identify this principal. For example, if you use UserInfo as Principal class, the id field maybe userId, userName, email, etc. For example, getUserId(), getUserName(), getEmail(), etc.
Default value is "id", that means your principal object has a method called "getId()"
```
大概意思就是UserInfo对象中必须要有id属性，并且要有对应的get方法。在身份验证的时候就已经将userinfo信息传递给了shiro，然后redis做缓存的时候需要key，key的值就与userinfo里面的id值有关。
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/5d634f7823c441ee80b9356c5024cce4.png)
点开UserInfo对象一看，尴了个尬，主键的命名使用的是uid
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/256e8900fd2c9cc5dfbb6629910f5775.png)
最后把主键换成id，就运行正常了。

演示项目在我的github上面，shiro-redis插件的整合可以查看：
[https://mrbird.cc/Spring-Boot-Shiro%20cache.html](https://mrbird.cc/Spring-Boot-Shiro%20cache.html)