---
title: SpringBoot整合Mail发送邮件&发送模板邮件
date: 2022-06-01 09:38:58
tags:
---

　　整合mail发送邮件，其实就是通过代码来操作发送邮件的步骤，编辑收件人、邮件内容、邮件附件等等。通过邮件可以拓展出短信验证码、消息通知等业务。

# 一、pom文件引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
<!--freemarker模板引擎是为了后面发送模板邮件 不需要的可以不引入-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```
# 二、application.yml文件中配置

```xml
spring:
  mail:
    host: smtp.qq.xyz #这里换成自己的邮箱类型 例如qq邮箱就写smtp.qq.com
    username: xx@qq.com #QQ邮箱
    password: xxxxxxxxxxx #邮箱密码或者授权码
    protocol: smtp #发送邮件协议
    properties.mail.smtp.auth: true
    properties.mail.smtp.port: 465 #端口号465或587
    properties.mail.smtp.starttls.enable: true
    properties.mail.smtp.starttls.required: true
    properties.mail.smtp.ssl.enable: true #开启SSL
    default-encoding: utf-8
freemarker:
  cache: false # 缓存配置 开发阶段应该配置为false 因为经常会改
  suffix: .html # 模版后缀名 默认为ftl
  charset: UTF-8 # 文件编码
  template-loader-path: classpath:/templates/  # 存放模板的文件夹，以resource文件夹为相对路径
```
邮箱密码暴露在配置文件很不安全，一般都是采取授权码的形式。点开邮箱，然后在账户栏里面点击生成授权码：
![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/c2d0d1f35a98ed2c68a4fdc2e120dbc6.png)

# 三、编写MailUtils工具类

```java
@Component
@Slf4j
public class MailUtils{

    /**
     * Spring官方提供的集成邮件服务的实现类，目前是Java后端发送邮件和集成邮件服务的主流工具。
     */
    @Resource
    private JavaMailSender mailSender;

    /**
     * 从配置文件中注入发件人的姓名
     */
    @Value("${spring.mail.username}")
    private String fromEmail;

    @Autowired
    private FreeMarkerConfigurer freeMarkerConfigurer;

    /**
     * 发送文本邮件
     * @param to      收件人
     * @param subject 标题
     * @param content 正文
     */
    public void sendSimpleMail(String to, String subject, String content) {
        SimpleMailMessage message = new SimpleMailMessage();
        //发件人
        message.setFrom(fromEmail);
        message.setTo(to);
        message.setSubject(subject);
        message.setText(content);
        mailSender.send(message);
    }

    /**
     * 发送html邮件
     */
    public void sendHtmlMail(String to, String subject, String content) {
        try {
            //注意这里使用的是MimeMessage
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            //第二个参数：格式是否为html
            helper.setText(content, true);
            mailSender.send(message);
        }catch (MessagingException e){
            log.error("发送邮件时发生异常！", e);
        }
    }

    /**
     * 发送模板邮件
     * @param to
     * @param subject
     * @param template
     */
    public void sendTemplateMail(String to, String subject, String template){
        try {
            // 获得模板
            Template template1 = freeMarkerConfigurer.getConfiguration().getTemplate(template);
            // 使用Map作为数据模型，定义属性和值
            Map<String,Object> model = new HashMap<>();
            model.put("myname","Ray。");
            // 传入数据模型到模板，替代模板中的占位符，并将模板转化为html字符串
            String templateHtml = FreeMarkerTemplateUtils.processTemplateIntoString(template1,model);
            // 该方法本质上还是发送html邮件，调用之前发送html邮件的方法
            this.sendHtmlMail(to, subject, templateHtml);
        } catch (TemplateException e) {
            log.error("发送邮件时发生异常！", e);
        } catch (IOException e) {
            log.error("发送邮件时发生异常！", e);
        }
    }

    /**
     * 发送带附件的邮件
     * @param to
     * @param subject
     * @param content
     * @param filePath
     */
    public void sendAttachmentsMail(String to, String subject, String content, String filePath) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            //要带附件第二个参数设为true
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(content, true);

            FileSystemResource file = new FileSystemResource(new File(filePath));
            String fileName = filePath.substring(filePath.lastIndexOf(File.separator));
            helper.addAttachment(fileName, file);
            mailSender.send(message);
        }catch (MessagingException e){
            log.error("发送邮件时发生异常！", e);
        }

    }
}
```
MailUtils其实就是进一步封装Mail提供的JavaMailSender类，根据业务场景可以在工具类里面添加对应的方法，这里提供了发送文本邮件、html邮件、模板邮件、附件邮件的方法。

# 四、Controller层的实现
```java
@Api(tags = "邮件管理")
@RestController
@RequestMapping("/mail")
public class MailController {

    @Autowired
    private MailUtils mailUtils;

    /**
     * 发送注册验证码
     * @return 验证码
     * @throws Exception
     */
    @ApiOperation("发送注册验证码")
    @GetMapping("/test")
    public String send(){
        mailUtils.sendSimpleMail("ruiyeclub@foxmail.com","普通文本邮件","普通文本邮件内容");
        return "OK";
    }

    /**
     * 发送注册验证码
     * @return 验证码
     * @throws Exception
     */
    @ApiOperation("发送注册验证码")
    @PostMapping("/sendHtml")
    public String sendTemplateMail(){
        mailUtils.sendHtmlMail("ruiyeclub@foxmail.com","一封html测试邮件",
                "<div style=\"text-align: center;position: absolute;\" >\n"
                        +"<h3>\"一封html测试邮件\"</h3>\n"
                        + "<div>一封html测试邮件</div>\n"
                        + "</div>");
        return "OK";
    }

    @ApiOperation("发送html模板邮件")
    @PostMapping("/sendTemplate")
    public String sendTemplate(){
        mailUtils.sendTemplateMail("ruiyeclub@foxmail.com", "基于模板的html邮件", "hello.html");
        return "OK";
    }

    @ApiOperation("发送带附件的邮件")
    @GetMapping("sendAttachmentsMail")
    public String sendAttachmentsMail(){
        String filePath = "D:\\projects\\springboot\\template.png";
        mailUtils.sendAttachmentsMail("xxxx@xx.com", "带附件的邮件", "邮件中有附件", filePath);
        return "OK";
    }
}
```
为了方便测试，这里使用了swagger3，详情可以查看[SpringBoot整合Swagger3生成接口文档](SpringBoot整合Swagger3生成接口文档)。

发送模板邮件这里，会读取resources下面的templates文件夹，测试中读取的是hello.html，具体代码如下：

```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8" />
    <title>freemarker简单示例</title>
</head>
<body>
<h1>Hello Freemarker</h1>
<div>My name is ${myname}</div>
</body>
</html>
```

# 五、测试结果

![image.png](https://ruiyeclub.oss-cn-shenzhen.aliyuncs.com/articles/ce977f2c1c55d0f5c49f0d863f0eda89.png)
如果需要达到通过邮件发送验证码的功能，可以使用redis。后台随机生成验证码，然后把用户的主键设为key，验证码的内容设为value，还可以设置个60s过期存储，发送成功后，用户登录通过主键从redis拿到对应的验证码，然后再进行登录验证就好了。

> 参考链接：[Spring Boot整合邮件配置](Spring Boot整合邮件配置)

GitHub地址：[https://github.com/ruiyeclub/SpringBoot-Hello](https://github.com/ruiyeclub/SpringBoot-Hello)