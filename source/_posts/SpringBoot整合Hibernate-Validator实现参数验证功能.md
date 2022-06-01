---
title: SpringBoot整合Hibernate Validator实现参数验证功能
date: 2022-06-01 09:34:37
tags:
---

　　在前后端分离的开发模式中，后端对前端传入的参数的校验成了必不可少的一个环节。但是在多参数的情况下，在controller层加上参数验证，会显得特别臃肿，并且会有许多的重复代码。这里可以引用Hibernate Validator来解决这个问题，直接在实体类进行参数校验，验证失败直接返回错误信息给前端，减少controller层的代码量。

# 一、pom引入Hibernate Validator

```xml
<!-- 验证器 -->
<dependency>
      <groupId>org.hibernate.validator</groupId>
      <artifactId>hibernate-validator</artifactId>
      <version>6.1.5.Final</version>
</dependency>
```
# 二、通过注解在实体类进行参数校验

```java
@Data
public class UserModel {

    @NotNull(message = "用户名称不能为空！")
    private String userName;

    @NotNull(message = "age不能为null!")
    @Range(min = 1, max = 888, message = "范围为1至888")
    private Integer age;

    /**
     * 日期格式化转换
     */
    @NotNull(message = "日期不能为null!")
    private Date date;
}
```
这里用到的参数校验的注解有@NotNull和@Range，message是到时候我们返回给前端的信息，注解的具体意思如下：
```java
@Null  被注释的元素必须为null
@NotNull  被注释的元素不能为null
@AssertTrue  被注释的元素必须为true
@AssertFalse  被注释的元素必须为false
@Min(value)  被注释的元素必须是一个数字，其值必须大于等于指定的最小值
@Max(value)  被注释的元素必须是一个数字，其值必须小于等于指定的最大值
@DecimalMin(value)  被注释的元素必须是一个数字，其值必须大于等于指定的最小值
@DecimalMax(value)  被注释的元素必须是一个数字，其值必须小于等于指定的最大值
@Size(max,min)  被注释的元素的大小必须在指定的范围内。
@Digits(integer,fraction)  被注释的元素必须是一个数字，其值必须在可接受的范围内
@Past  被注释的元素必须是一个过去的日期
@Future  被注释的元素必须是一个将来的日期
@Pattern(value) 被注释的元素必须符合指定的正则表达式。
@Email 被注释的元素必须是电子邮件地址
@Length 被注释的字符串的大小必须在指定的范围内
@NotEmpty  被注释的字符串必须非空
@Range  被注释的元素必须在合适的范围内
```
# 三、controller层的方法加上@Valid注解

```java
@PostMapping("/testPost")
public Object testPost(@RequestBody @Valid UserModel userModel, BindingResult result){
    if(result.hasErrors()){
        for(ObjectError error:result.getAllErrors()){
            return error.getDefaultMessage();
        }
    }
    return userModel;
}
```
controller层这里只需要在实体类的前面加上@Valid注解，这个注解可以实现数据的验证。这里BindingResult是存储了校验时的错误信息，验证有误时将错误信息返回给前端。这里不使用BindingResult的时候，控制台会报MethodArgumentNotValidException，这里可以通过自定义异常类来捕捉它，然后去掉BindingResult，以及难看的if判断。

# 四、自定义异常类捕捉MethodArgumentNotValidException

```java
@RestControllerAdvice
public class GlobalExceptionAdvice {

    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public JsonData validException(MethodArgumentNotValidException e) {
        //验证post请求的参数合法性
        MethodArgumentNotValidException notValidException = e;
        String msg = notValidException.getBindingResult().getFieldError().getDefaultMessage();
        return JsonData.buildError(msg);
    }
}
```
使用PostMan的测试结果如下：
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/0e7da5f5ea00deaf1e69196d09085f49.png)

具体的代码可以在我的github上面查看，https://github.com/ruiyeclub/SpringBoot-Hello
