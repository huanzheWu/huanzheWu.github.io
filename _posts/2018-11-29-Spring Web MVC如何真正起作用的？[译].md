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
在本文中，我们将使用最新的，也是最好的版本Spring Frameword 5.我们将重点放在Spring经典的Web栈上，从框架的最初版本开始，它仍然是构建Web应用程序的主要方式。对于初学者来说，你需要使用Spring Boot以及一些starter依赖包来构建你的测试项目：
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

为了方便Controller 方法可以同事返回这两种对象，将它们合并到了ModelAndView类型中。为了呈现出HTML页面，我们使用Thymeleaf作为视图模版引擎，该引擎集成于Spring中，可靠、开箱即用。

## Servlet作为Java Web应用程序的基础
那么，当我们在浏览器中输入http：// localhost：8080 /时，实际会发生什么呢？按Enter键，请求会到达Web服务器？ 我们如何从请求中获取在浏览器中提交的Web表单信息？鉴于本项目是一个简单的Spring Boot应用程序，我们能够通过Spring5Application运行它。Spring Boot默认使用Apache Tomcat。 因此，运行应用程序时，可能会在日志中看到以下信息：
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
由于Tomcat是一个Servlet容器，因此，发送到Tomcat Web服务器的每个HTTP请求自然都会由Java servlet处理。 Spring Web应用程序入口点是一个servlet毫不奇怪。简而言之，servlet是任何Java Web应用程序的核心组件; 它是低级别的，并没有对特定编程模式（如MVC）施加太多限制。HTTP servlet只能接收HTTP请求，以某种方式处理它，然后发回响应。而且，从Servlet 3.0 API开始，可以使用基于Java的配置来代替基于XML的配置。

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
首先，我们来跟踪一个简单的请求从到达控制层的方法到返回到浏览器客户端的过程。DispatcherServlet类有着很深的继承层次结构。从上到下逐一地理解这些继承层次是值得的。请求处理方法是我们最感兴趣的部分。
[1.png](https://postimg.cc/18sWYqkc)

[![1.png](https://i.postimg.cc/wjsSM5FP/1.png)]()

## GenericServlet

GenericServlet是Servlet规范的一部分，它并没有直接关注HTTP。 它定义了接收传入请求并生成响应的service（）方法。请注意ServletRequest和ServletResponse方法参数如何与HTTP协议无关：

```java
public abstract void service(ServletRequest req, ServletResponse res) 
  throws ServletException, IOException;
```

请求到达server时，最终会调用到该service方法，包括最简单的GET请求。

## HttpServlet
顾名思义，HttpServlet类是以HTTP为中心的Servlet实现，也是由规范定义的。在更实际的术语中，HttpServlet是一个带有service（）方法实现的抽象类，它通过HTTP方法类型拆分请求，看起来大致如下：
```java
protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException {

    String method = req.getMethod();
    if (method.equals(METHOD_GET)) {
        // ...
        doGet(req, resp);
    } else if (method.equals(METHOD_HEAD)) {
        // ...
        doHead(req, resp);
    } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);
        // ...
    }
```

## HttpServletBean
HttpServletBean是层次结构中第一个支持Spring的类。 它使用从web.xml或WebApplicationInitializer接收的servlet init-param值注入bean的属性。如果对我们的应用程序发出请求，则会针对这些特定的HTTP请求调用doGet（），doPost（）等方法。


## FrameworkServlet
FrameworkServlet将Servlet功能与Web应用程序上下文集成，实现ApplicationContextAware接口。 但它也能够自己创建Web应用程序上下文。正如我们已经看到的，HttpServletBean超类将init-params注入为bean属性。 因此，如果在servlet的contextClass init-param中提供了上下文类名，那么将创建此类的实例作为应用程序上下文。 否则，将使用默认的XmlWebApplicationContext类。

由于XML配置现在不合时宜，Spring Boot默认使用AnnotationConfigWebApplicationContext配置DispatcherServlet。 但你可以很容易地改变它。例如，如果需要使用基于Groovy的应用程序上下文配置Spring Web MVC应用程序，则可以在web.xml文件中使用以下DispatcherServlet配置：
```java
    dispatcherServlet
        org.springframework.web.servlet.DispatcherServlet
        contextClass
 org.springframework.web.context.support.GroovyWebApplicationContext
```

可以使用WebApplicationInitializer类以更现代的基于Java的方式完成相同的配置。

## DispatcherServlet：统一请求处理
HttpServlet.service（）实现以HTTP动词的类型路由请求，在低级servlet的上下文中非常有意义。 但是，在Spring MVC抽象级别，方法类型只是可用于将请求映射到其处理程序的参数之一。因此，FrameworkServlet类的另一个主要功能是将处理逻辑连接回单个processRequest（）方法，该方法又调用doService（）方法：
```java
@Override
protected final void doGet(HttpServletRequest request, 
  HttpServletResponse response) throws ServletException, IOException {
    processRequest(request, response);
}

@Override
protected final void doPost(HttpServletRequest request, 
  HttpServletResponse response) throws ServletException, IOException {
    processRequest(request, response);
}

// …
```

## DispatcherServlet：丰富请求
最后，DispatcherServlet实现了doService（）方法。 在这里，它向请求添加了一些有用的对象，它们可以在处理管道中派上用场：Web应用程序上下文，区域设置解析器，主题解析器，主题源等：
```java
request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, 
  getWebApplicationContext());
request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
```
此外，doService（）方法准备输入和输出flash映射。 Flash映射基本上是一种模式，用于将参数从一个请求传递到紧随其后的另一个请求。 这在重定向期间可能非常有用（例如在重定向后向用户显示一次性信息消息）：
```java
FlashMap inputFlashMap = this.flashMapManager
  .retrieveAndUpdate(request, response);
if (inputFlashMap != null) {
    request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, 
      Collections.unmodifiableMap(inputFlashMap));
}
request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
```
然后，doService（）方法调用负责请求分派的doDispatch（）方法。

## DispatcherServlet：调度请求
dispatch（）方法的主要目的是为请求找到合适的处理程序并为其提供请求/响应参数。 处理程序基本上是任何类型的Object，并不限于特定的接口。 这也意味着Spring需要为这个知道如何与处理程序“交谈”的处理程序找到一个适配器。

为了找到与请求匹配的处理程序，Spring浏览了HandlerMapping接口的已注册实现。 有许多不同的实现可以满足您的需求。

SimpleUrlHandlerMapping允许通过其URL将请求映射到某个处理bean。 例如，可以通过使用类似于此的java.util.Properties实例注入其mappings属性来配置它：
```java
/welcome.html=ticketController
/show.html=ticketController
```
处理程序映射最广泛使用的类可能是RequestMappingHandlerMapping，它将请求映射到@Controller类的@ RequestMapping-annotated方法。 这正是将调度程序与控制器的hello（）和login（）方法连接起来的映射。

请注意，我们的方法相应地使用@GetMapping和@PostMapping进行注释。 反过来，这些注释用@RequestMapping元注释标记。dispatch（）方法还负责其他一些特定于HTTP的任务：

- 在未修改资源的情况下，对GET请求进行短路处理
- 将多部分解析器应用于相应的请求
- 如果处理程序选择异步处理请求，则对请求进行短路处理

## 处理请求
