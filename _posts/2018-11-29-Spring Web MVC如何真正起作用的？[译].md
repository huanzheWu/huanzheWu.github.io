---
layout:     post
title:      Spring Web MVC如何真正起作用的？[译]
subtitle:   How Spring Web MVC Really Works
date:       2018-11-29
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spring MCV
---

## 前言
本文是对Spring Web MVC强大特性与内工作原理的深入研究，它是Spring Framework的一部分。在GitHub上提供了本文的源代码。

## 项目设置
在本文中，我们将使用最新的，也是最好的版本Spring Frameword 5.我们将重点放在Spring经典的Web栈上，从框架的最初版本开始，它仍然是构建Web应用程序的主要方式。

对于初学者来说，你需要使用Spring Boot以及一些starter依赖包来构建你的测试项目：
```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.M5</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
</dependencies>
```
注意，为了使用Spring 5，您还需要使用Spring Boot 2.x. 在撰写本文时，这是一个里程碑版本，可在Spring Milestone Repository中找到。 让我们将此存储库添加到您的Maven项目中：
```java
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```
你可以在Maven Central上查看当前版本的Spring Boot。

## 一个简单的项目
我们将实现一个简单的应用程序，该应用程序是一个登录的页面。通过这个简单的项目，我们来了解Srping Web MVC的工作原理。要显示登录页面，我们需要创建一个使用@Controller 注解的类，它接收根目录上的GET请求。

下面代码中的`hello()`方法是个无参方法。它返回了一个字符串，该字符串将被Spring MVC解析为视图的名称。在我们的例子中，这个视图为`login.html`：
```java
import org.springframework.web.bind.annotation.GetMapping;

@GetMapping("/")
public String hello() {
    return "login";
}
```
为了处理用户登录时提交的信息，我们需要创建另外的一个方法，来处理POST请求。依据处理的逻辑，该方法会将用户的登录结果（成功或失败）重定向到成功或失败的页面。

需要注意的是，`login()`方法接收域对象作为参数，并返回ModelAndView对象：
```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.servlet.ModelAndView;

@PostMapping("/login")
public ModelAndView login(LoginData loginData) {
    if (LOGIN.equals(loginData.getLogin()) 
      && PASSWORD.equals(loginData.getPassword())) {
        return new ModelAndView("success", 
          Collections.singletonMap("login", loginData.getLogin()));
    } else {
        return new ModelAndView("failure", 
          Collections.singletonMap("login", loginData.getLogin()));
    }
}
```
`ModelAndView`是两个不同对象的持有者：
- 模型 - 用于呈现页面的数据的键值映射
- 视图 - 填充了模型数据的页面模板

为了方便Controller 方法可以同事返回这两种对象，将它们合并到了ModelAndView类型中。

为了呈现出HTML页面，我们使用Thymeleaf作为视图模版引擎，该引擎集成于Spring中，可靠、开箱即用。

## Servlet作为Java Web应用程序的基础
那么，当我们在浏览器中输入http：// localhost：8080 /时，实际会发生什么呢？按Enter键，请求会到达Web服务器？ 我们如何从请求中获取在浏览器中提交的Web表单信息？

鉴于本项目是一个简单的Spring Boot应用程序，我们能够通过Spring5Application运行它。

Spring Boot默认使用Apache Tomcat。 因此，运行应用程序时，可能会在日志中看到以下信息：
```java
2017-10-16 20:36:11.626  INFO 57414 --- [main] 
  o.s.b.w.embedded.tomcat.TomcatWebServer  : 
  Tomcat initialized with port(s): 8080 (http)

2017-10-16 20:36:11.634  INFO 57414 --- [main] 
  o.apache.catalina.core.StandardService   : 
  Starting service [Tomcat]

2017-10-16 20:36:11.635  INFO 57414 --- [main] 
  org.apache.catalina.core.StandardEngine  : 
  Starting Servlet Engine: Apache Tomcat/8.5.23
```
由于Tomcat是一个Servlet容器，因此，发送到Tomcat Web服务器的每个HTTP请求自然都会由Java servlet处理。 Spring Web应用程序入口点是一个servlet毫不奇怪。

简而言之，servlet是任何Java Web应用程序的核心组件; 它是低级别的，并没有对特定编程模式（如MVC）施加太多限制。

HTTP servlet只能接收HTTP请求，以某种方式处理它，然后发回响应。

而且，从Servlet 3.0 API开始，可以使用基于Java的配置来代替基于XML的配置。

## DispatcherServlet作为Spring MVC的核心
作为Web应用程序的开发人员，我们真正想做的将乏味和样板任务流程抽象出来，让我们能够专注于有用的业务逻辑：

- 将HTTP请求映射到某种处理方法
- 将HTTP请求数据和标头解析为数据传输对象（DTO）或域对象
- 模型 - 视图 - 控制器交互
- 生成DTO，域对象等的响应

Spring DispatcherServlet就是提供了这样的能力。 它是Spring Web MVC框架的核心; 此核心组件接收对的应用程序的所有请求。正如后面将看到的，DispatcherServlet是强可扩展的。 例如，它允许为许多任务插入不同的、已有的或新适配器：

- 将请求映射到一个类或方法上（HandlerMapping接口的实现），让对应的类或方法来处理请求。
- 使用特定的模式处理请求，比如常规的servlet，或更加复杂的MVC工作流，或只是POJO bean中的方法（HandlerAdapter接口的实现）
- 按名称解析视图，允许使用不同的模板引擎，XML，XSLT或任何其他视图技术（ViewResolver接口的实现）
- 使用默认的Apache Commons文件上传实现或编写自己的MultipartResolver来解析多部分请求
- 使用任何LocaleResolver实现解析语言环境，包括cookie，会话，Accept HTTP标头或确定用户期望的语言环境的任何其他方式

## 处理HTTP请求


![](http://ww2.sinaimg.cn/large/7853084cgw1fa3cqnu8s2g207i0dc4qp.gif)


