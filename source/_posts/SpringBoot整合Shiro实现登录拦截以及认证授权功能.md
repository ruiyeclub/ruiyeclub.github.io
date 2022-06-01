---
title: SpringBoot整合Shiro实现登录拦截以及认证授权功能
date: 2022-02-11 17:28:11
tags:
---

## 一、Shiro是什么？

Apache Shiro是一个Java安全权限框框架。

Shiro可以非常容易的开发出足够好的应用，其不仅可以在javaEE环境。

Shiro可以完成，认证，授权，加密，会话管理，Web集成，缓存等。

## 二、Shiro工作原理
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/eb29de36230aeb29dcc9257948081472.png)
应用代码的交互对象是 “Subject”，该对象代表了**当前 “用户”**，而所有用户的安全操作都会交给 SecurityManager 来管理，而管理过程中会从 Realm 中获取用户对应的角色和权限，可以把 Realm 堪称是安全数据源。
 
也就是说，我们要使用最简单的 Shiro 应用：
- 通过 Subject 来进行认证和授权，而 Subject 又委托给了 SecurityManager 进行管理
- 我们需要给 SecurityManager 注入 Realm 以便其获取用户和权限进行判断
- 也即，Shiro 不提供用户和权限的维护，需要由开发者自行通过 Realm 注入

## 三、SpringBoot整合Shiro

1.xml导入jar包
```xml
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.4.1</version>
        </dependency>
```
2.config目录下创建ShiroConfig.java，拦截以及授权等功能都在这里配置
```java
@Configuration
public class ShiroConfig {
    //1.创建realm对象 需要自定义
    @Bean
    public UserRealm userRealm(){
        return new UserRealm();
    }

    //2.DefaultWebSecurityManager
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(@Qualifier("userRealm") UserRealm userRealm){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //关联UserRealm
        securityManager.setRealm(userRealm);
        return securityManager;
    }

    //3.ShiroFilterFactoryBean
    @Bean
    public ShiroFilterFactoryBean getShiroFilterFactoryBean(@Qualifier("securityManager")DefaultWebSecurityManager defaultWebSecurityManager){
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        //设置安全管理器
        bean.setSecurityManager(defaultWebSecurityManager);

        //添加shiro的内置过滤器
        /**
         * anon：无需认证就可以访问
         * authc：必须认证了才能访问
         * user：必须拥有 记住我 功能才能用
         * perms：拥有对某个资源的权限才能访问
         * role：拥有某个角色权限才能访问
         */
        Map<String, String> filterMap = new LinkedHashMap<>();
        //授权
        filterMap.put("/user/add","perms[user:add]"); //进入需要授权（授权规则用户后面接：add）才可以进入add
        filterMap.put("/user/update","perms[user:update]");

        //拦截功能
        filterMap.put("/user/*","authc"); //表示访问user接口的资源都要认证
        bean.setFilterChainDefinitionMap(filterMap);

        //******处理权限不够或者需要授权的业务********
        //如果没有权限authc 设置登录的请求
        bean.setLoginUrl("/toLogin");
        //未授权页面
        bean.setUnauthorizedUrl("/noauth");
        return bean;
    }
}
```
3.自定义UserRealm类，继承AuthorizingRealm重写用户授权和用户认证的方法
```java
public class UserRealm extends AuthorizingRealm {//用户授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {

        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
//        info.addStringPermission("user:add"); 手动添加了权限

        //拿到当前登录的这个对象
        Subject subject= SecurityUtils.getSubject();
        User currentUser = (User) subject.getPrincipal(); //拿到user对象 可以设置用户权限
        //设置当前用户的权限 从数据库上面拿
        info.addStringPermission(currentUser.getPerms());
        System.out.println(currentUser.getPerms());
        return info;
    }

    //用户认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        UsernamePasswordToken userToken = (UsernamePasswordToken) token;
        //认证用户（连接数据库）
        User user = userService.queryUserByName(userToken.getUsername());
        if(user==null){
            return null; //UnknownAccountException
        }

        //判断session是否有值，显示登录按钮
        Subject currentSubject = SecurityUtils.getSubject();
        Session session = currentSubject.getSession();
        session.setAttribute("loginUser",user);

        //可以加密：MD5 MD5盐值加密（更高级）
        //密码认证（shiro完成）
        return new SimpleAuthenticationInfo(user,user.getPassword(),"");
    }
}
```
Shiro大致的配置就在这里了，具体功能如拦截功能、授权认证功能，可以在我的
github [https://github.com/ruiyeclub/SpringBoot-Hello](https://github.com/ruiyeclub/SpringBoot-Hello)，进行查看。

Shiro核心概述可参考文章：[https://www.cnblogs.com/deng-cc/p/9401900.html](https://www.cnblogs.com/deng-cc/p/9401900.html)