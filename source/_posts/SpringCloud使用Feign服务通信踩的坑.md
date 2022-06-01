---
title: SpringCloud使用Feign服务通信踩的坑
date: 2022-06-01 09:39:35
tags:
---

　　**fallback熔断器实现了Feign客户端的所有方法，当网络不通或者访问失败时，会自动调用fallback服务降级类中的方法。**

启动项目时报错了，具体的报错信息如下：

```java
Caused by: java.lang.IllegalStateException: No fallback instance of type class com.xxx.xxx.feign.fallback.RemoteUserFallback found for feign client xxx
```

报错内容明显是没找到RemoteUserFallBack这个类

1、检查配置文件

```xml
feign:
  hystrix:
    enabled: true # 开启Feign的熔断功能 默认是关闭的
```
2、启动类上需要@EnableFeignClients注解

```java
@EnableFeignClients(basePackages = {"com.xxx.包名"}) //开启Feign并扫描Feign客户端
```

3、Feign客户端类上使用@FeignClient，通过fallback属性来指明对应熔断器的类名

```java
@FeignClient(value = "服务名", fallback = RemoteUserFallback.class,) //声明当前类是一个Feign客户端，并指定请求的服务名
```
**4、fallback熔断器类上需要加注解@Component，确保可以被spring扫描**

我报错的原因就是出现在第四步这里，尽管我加了@component注解。SpringBoot在启动的时候 会扫描main类所在包及其子包进行Bean的实例化，但是fallback熔断器类并不在我启动类的子类下面，我这里是通过引入其模块来调用这里面的方法。

所以最后我在启动类上加了@ComponentScan注解：

```java
@ComponentScan(basePackages = {"com.xxx"})
```
OK，成功启动并访问成功。