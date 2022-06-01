---
title: 启动项目报错Correct the classpath of your application so that it contains a single, compatible version of xxx
date: 2022-02-13 17:28:11
tags:
---

项目是基于Gradle构建的，在整合swagger后，启动项目时报错了。报错日志：

```java
Description:

An attempt was made to call a method that does not exist. The attempt was made from the following location:

    springfox.documentation.spring.web.plugins.DocumentationPluginsManager.createContextBuilder(DocumentationPluginsManager.java:152)

The following method did not exist:

    'org.springframework.plugin.core.Plugin org.springframework.plugin.core.PluginRegistry.getPluginFor(java.lang.Object, org.springframework.plugin.core.Plugin)'

The method's class, org.springframework.plugin.core.PluginRegistry, is available from the following locations:

    jar:file:/C:/Users/Administrator/.gradle/caches/modules-2/files-2.1/org.springframework.plugin/spring-plugin-core/2.0.0.RELEASE/95fc8c13037630f4aba9c51141f535becec00fe6/spring-plugin-core-2.0.0.RELEASE.jar!/org/springframework/plugin/core/PluginRegistry.class

It was loaded from the following location:

    file:/C:/Users/Administrator/.gradle/caches/modules-2/files-2.1/org.springframework.plugin/spring-plugin-core/2.0.0.RELEASE/95fc8c13037630f4aba9c51141f535becec00fe6/spring-plugin-core-2.0.0.RELEASE.jar


Action:

Correct the classpath of your application so that it contains a single, compatible version of org.springframework.plugin.core.PluginRegistry
```
百度之后，发现是jar包冲突了，导入了两个不同版本的jar包。如图：
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/a3688f4401c04f3a53efb72e701a4457.png)
解决办法可以直接将依赖中的jar包剔除掉，或者直接删除该依赖也行。
