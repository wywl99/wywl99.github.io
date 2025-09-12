---
title: 'SpringBoot参数校验'
publishDate: 2022-05-13
updatedDate: 2025-02-24
description: '3D imagery has the power to bring cinematic visions to life and help accurately plan tomorrow’s cityscapes. Here, 3D expert Ricardo Ortiz explains how it works.'
tags:
  - Java
  - Spring
language: '简体中文'
heroImage: { src: './thumbnail.jpg', color: '#D58388' }
---

# Java 参数校验

参数校验是开发一个健壮程序很重要的步骤，如果对前端传过来的参数不进行校验，轻则导致单个用户数据出错，重则可能导致系统被攻击。有一句至理名言：**永远不要相信客户端传过来的参数**

但是参数校验又是一个很容易忽视的步骤，传统的参数校验一般都是在项目中写大量的 if else 去做判断，这样容易使得代码臃肿，干扰到阅读业务逻辑。且手写大量的判断本身也是一个体力活，很多喜欢偷懒的（比如我）就容易忽略掉参数的校验

但是最近发现了一个挺好用的工具 `Hibernate Validator` ，可以在 Bean 上添加注解进行参数校验，而且搭配 SpringBoot 可以很方便的返回参数错误信息

## 环境

项目环境是 `SpringBoot 2.6.7 Maven 构建`

在 SpringBoot 的 Pom 文件中导入

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```



## 使用

创建一个实体类，在需要进行校验的字段上添加校验条件和错误说明。

```java
@Data
public class User implements Serializable {

    @NotNull(message = "id不能为空")
    private Long id;

    @NotNull(message = "用户名不能为空")
    private String username;

    @Length(min = 8,max = 16,message = "密码长度只能在8-16位之间")
    private String password;

    /**
     * 男0 女1
     */
    @Range(min = 0,max = 1,message = "性别设置错误")
    private Integer sex;

    @Max(value = 100L,message = "年龄设置错误")
    private Integer age;
}

```

例如 `@NotNull(message = "id不能为空")` 就代表被修饰的字段不能为空，如果为空的话就会报错，错误信息就是后面的 `message`

+ `@Length` 限制字符串长度

+ `@Range` 限制取值的区间

+ `@Max` 限制最大取值

这里只使用了 Hibernate Validator 中一小部分注解，还有很多其他的注解可以在文末的官方文档中找到



继续创建一个 Controller

```java
@RestController
public class ValidationController {
    @PostMapping("/validation")
    public String test(@RequestBody @Validated User user){
        System.out.println(user);
        return "ok";
    }
}
```

向 `/validation` 发送请求的时候，就会对 user 接受到的参数进行校验。**切记一定要在被校验的参数前加上`@Validated` 注解，否则不会生效**



这里发送请求，将参数 id 设置为 null。

![image.png](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2022/05/image-17af7cabae774e58bd72a7506b2d7dbf.png)

User 这个类的字段 id 是添加了 @NotNull 注解的，所以这里这里报异常了，有个 default message 提示 id 不能为空，还有下面的 username 我们也是添加了 @NotNull 注解的，也出现了 message 中的提示。




## 错误信息格式化

虽然校验是有了，但是错误信息实在太多了，没有可阅读性。我们也不能将这一大段错误信息全部返回前端，通常只返回错误提示就够了，如果有多个就用逗号拼接。这里可以结合 spring mvc 的全局异常处理来实现。



创建一个全局异常处理器来处理参数校验失败后给客户端的提示信息，Hibernate Validator 会将错误信息封装到 `MethodArgumentNotValidException` 中，我们需要做的就是将我们自定义的 message 取出来

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public String handleMethodArgumentException(MethodArgumentNotValidException exception){
        return exception.getBindingResult().getAllErrors().stream().map(DefaultMessageSourceResolvable::getDefaultMessage).collect(Collectors.joining(","));
    }
}
```

这里的逻辑就是获取到异常类中的所有错误信息用逗号拼接起来。因为可能有多个参数校验不通过，所以这里返回的是一个集合，用 stream 操作了一下



然后再发送请求，就返回了我们自定义的提示语

![image.png](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2022/05/image-51ff4ecfec234ca6a641054074aeffdf.png)



这里演示的比较简单，项目中一般会有通用的返回结构体，可以将错误信息封装进去

![image.png](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2022/05/image-8ca47f62c725484390628537de488941.png)



## 官方文档

Hibernate Validator 官方文档：https://hibernate.org/validator/documentation/
