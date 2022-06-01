---
title: SpringBoot使用Jwt处理跨域认证问题
date: 2022-02-14 17:35:40
tags:
---

　　在前后端开发时为什么需要用户认证呢？原因是由于HTTP协定是不存储状态的，这意味着当我们透过账号密码验证一个使用者时，当下一个request请求时他就把刚刚的资料忘记了。于是我们的程序就不知道谁是谁了。 所以为了保证系统的安全，就需要验证用户是否处于登陆状态。

# 一、JWT的组成

JWT由Header、Payload、Signature三部分组成，分别用.分隔。

下面就是一个jwt真实的样子，说白了就是一个字符串，但是里面却存储了很重要的信息。

```java
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJyYXljaGVuIiwiaWQiOjIsIm5hbWUiOiJyYXkiLCJwYXNzd29yZCI6IjMyMSIsImlhdCI6MTU5MDI5OTU0OCwiZXhwIjoxNTkwMzg1OTQ4fQ.ORJNldDIfffg7D3_xu0_dBWb16y4fPLtw_r6qgScFpQ
```
Header：
> 第一部分是请求头由两部分组成，alg与typ,第一个指定的是算法，第二指定的是类型。

Payload
> 第二部分是主体信息组成,用来存储JWT基本信息，或者是我们的信息。

Signature
> 第三部分主要是给第一部分跟第二部进行签名使用的,用来验证是否是我们服务器发起的Token，secret是我们的密钥。

# 二、在springboot项目中使用jwt做验证

具体流程：

-  把用户的用户名和密码发到后端
-  后端进行校验，校验成功会生成token, 把token发送给客户端
-  客户端自己保存token, 再次请求就要在Http协议的请求头中带着token去访问服务端，和在服务端保存的token信息进行比对校验。

1.先引入jar包
```xml
<dependency>
   <groupId>io.jsonwebtoken</groupId>
   <artifactId>jjwt</artifactId>
   <version>0.9.1</version>
</dependency>
```
2.使用工具类生成token和验证token（生成token方法中存入了用户的信息）

```java
public class JwtUtils {

    //发行者
    public static final String SUBJECT="raychen";
    //过期时间 一天
    public static final long EXPIRE=1000*60*60*24;
    //密钥
    public static final String APPSECRET="raychen11";

    /**
     * 生成jwt
     * @param user
     * @return
     */
    public static String geneJsonWebToken(User user){
        if(user==null || user.getId()==null || user.getName()==null || user.getPassword()==null){
            return null;
        }
        String token=Jwts.builder().setSubject(SUBJECT)
                .claim("id",user.getId())
                .claim("name",user.getName())
                .claim("password",user.getPassword())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis()+EXPIRE))
                .signWith(SignatureAlgorithm.HS256,APPSECRET).compact();

        return token;
    }

    /**
     * 校验token
     * @param token
     * @return
     */
    public static Claims checkJWT(String token){
        //仿造的token或者已过期就会报错
        try {
            final Claims claims=Jwts.parser().setSigningKey(APPSECRET).parseClaimsJws(token).getBody();
            return claims;
        }catch (Exception e){
            System.out.println("catch...");
        }
        return null;
    }
}
```
3.自定义注解（进行token验证）
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckToken {
    boolean required() default true;
}
```
4.编写config，将后台所有请求先去拦截器（拦截器返回了true，用户才可以请求到接口）
```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {


    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authenticationInterceptor())
                .addPathPatterns("/**");    // 拦截所有请求，通过判断是否有 @LoginRequired 注解 决定是否需要登录
    }

    @Bean
    public AuthenticationInterceptor authenticationInterceptor() {
        return new AuthenticationInterceptor();
    }
}
```
5.定义拦截器（对需要token验证的请求，进行验证，验证成功返回true，失败返回false无法请求到接口）
```java
public class AuthenticationInterceptor implements HandlerInterceptor {

    public static final Gson gson=new Gson();
    /**
     * 进入controller之前进行拦截
     * @param request
     * @param response
     * @param object
     * @return
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object object) throws Exception {
        // 如果不是映射到方法直接通过
        if (!(object instanceof HandlerMethod)) {
            return true;
        }

        HandlerMethod handlerMethod = (HandlerMethod) object;
        Method method = handlerMethod.getMethod();
        //检查是否有LoginToken注释，有则跳过认证
        if (method.isAnnotationPresent(LoginToken.class)) {
            LoginToken loginToken = method.getAnnotation(LoginToken.class);
            if (loginToken.required()) {
                return true;
            }
        }

        //前面是不需要token验证的 从 http 请求头中取出 token
        String token = request.getHeader("token");
        //检查有没有需要用户权限的注解
        if (method.isAnnotationPresent(CheckToken.class)) {
            CheckToken checkToken = method.getAnnotation(CheckToken.class);
            if (checkToken.required()) {
                if(token!=null){
                    Claims claims= JwtUtils.checkJWT(token);
                    if(null==claims){
                        sendJsonMessage(response, JsonData.buildError("token有误"));
                        return false;
                    }
                    Integer userId= (Integer) claims.get("id");
                    String name = (String) claims.get("name");

                    request.setAttribute("userId",userId);
                    request.setAttribute("name",name);
                    return true;
                }
                //token为null的话 返回一段json给前端
                sendJsonMessage(response, JsonData.buildError("请登录"));
                return false;
            }
        }
        //没有使用注解的方法 直接返回true
        return true;
    }

        /**
         * 响应数据给前端
         * @param response
         * @param obj
         * @throws IOException
         */
        public static void sendJsonMessage (HttpServletResponse response, Object obj) throws IOException {
            response.setContentType("application/json; charset=utf-8");
            PrintWriter writer = response.getWriter();
            writer.print(gson.toJson(obj));
            writer.close();
            response.flushBuffer();
        }
}
```
用户登录成功后，使用工具类生成token。token在服务端不做存储，直接将token返回给客户端，客户端下次请求服务端时，使用工具类来验证header里的token是否合法。

项目代码地址：[https://github.com/ruiyeclub/SpringBoot-Hello](https://github.com/ruiyeclub/SpringBoot-Hello)
