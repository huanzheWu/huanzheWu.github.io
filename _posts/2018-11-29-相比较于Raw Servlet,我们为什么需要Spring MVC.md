---
layout:     post
title:      相比较于原生的servlet，我们为什么需要Spring MCV?
subtitle:   
date:       2018-11-29
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spring MCV
​---

相比较于原生的servlet，我们为什么要使用Spring MVC 或Structs 等框架来开发Web应用？这个问题在Stackoverflow上有网友进行了讨论，下面是一些论点：

## 可拓展，符合最佳实践

>If you're building a really quick and dirty demo that you have no intention of extending later, spring can result in a lot of additional configuration issues (not really if you've done it before, but I always end up fighting with it one way or another), so that might be a time to consider just using plain old servlets. Generally though, anything beyond just a super fast and dirty demo, using some form of MVC framework is going to make life in the future a lot easier and is also in line with best practices. Spring makes things super easy, just have to spend some time on the front end configuring everything.I should note, there's nothing you can do with java servlets that you can't do with Spring. The big difference is setup time.
>


如果是要开发一个非常简单的、不需考虑以后拓展的demo程序，那么可以使用原生的servlet，这时使用MVC框架，反而会因为配置上的问题，引入不必要的复杂性。除此之外，使用MVC框架将使得开发简便。

## 主要还是方便：框架提供了很多通用的支持

> Servlet technology is used for more generic server side extension for request-response paradigm. And Spring just uses it for the Web application over HTTP.
> 


Servlet基于用于处理请求和响应的低级API。像Spring MVC这样的Web框架旨在使构建Web应用程序变得更加容易，这些应用程序可以更轻松地处理HTTP请求和响应。大多数Java Web框架（包括Spring MVC）都在后台使用servlet。

可以使用servlet编写Web应用程序，但您必须手动处理所有详细信息。对于典型的Web内容，例如验证，REST，JSON的请求/响应主体，表单绑定等，将得到很少的帮助。您最终将编写大量实用程序代码来支持Web应用程序。

另一方面，Web框架旨在使所有这些内容变得简单。使用Spring MVC，可以手动处理请求和响应，即使仍然可以在需要时访问它们。想在Spring MVC中返回JSON吗？只需添加@ResponseBody注释，Spring就会附加它。想要RESTful URL吗？简单。输入验证？小菜一碟。想要将表单数据绑定到对象？简单。使用servlet，必须手动完成所有这些操作。

## 当然，使用原生的servlet，你可以懂得更多

> However, code with raw api and servlet gives you chance to gain experience and be a programmer.
> 

使用原始api和servlet的代码可以让您获得经验并成为更厉害的程序员。

## 也有人说，servlet已经足够使用了

> I don't use Spring a lot. But I don't see how would it have big impact on the performance. MVC's can help, but they can create a mess and extra work and frustration. The old good way is good enough for most projects implemented by one programmer. MVC's could help when there are more than one developer.
> 

使用MVC虽然可以提供帮助，但是它们同样可以造成混乱、额外的工作和挫败感，旧的方法已经足够了。当有多个开发人员的时侯，MVC或许可以提供帮助。

## 现代开发需要IOC

和同事闲聊的时候谈到现代的开发运维模式需要IOC，运维的同学可以通过修改配置文件来影响程序，而不必去改代码重新打包。基于配置文件的IOC刚好可以满足这一点。




