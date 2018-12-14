---
layout:     post
title:      Spring MVC(二) 使用HandlerMethodArgumentResolver来进行参数解析
subtitle:   如何把请求的参数，解析到Bean中？
date:       2018-12-14
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spring MCV
---



# 在Controller中解析请求参数

来看一个场景。

我们的工程代码中有一个bean，定义了`Student`学生的相关属性，`Student`类包括了两个字段用来描述学生姓名和班级：

```java
package org.springframework.samples.mvc.data.bean;

public class Student {
    public String name;
    public int classNum;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getClassNum() {
        return classNum;
    }

    public void setClassNum(int classNum) {
        this.classNum = classNum;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", classNum=" + classNum +
                '}';
    }
}
```

现在我们想提交`Student`的`name`和`classNum`。我们需要定义一个`Controller`如下：

``` java
@Controller
public class AddStudentInfoController {

    @RequestMapping("/data/addStudent")
    public String  addStudent(HttpServletRequest httpServletRequest){
        String name = httpServletRequest.getParameter("name");
        String classNum = httpServletRequest.getParameter("classNum");
        Student student = new Student();
        student.setName(name);
        student.setClassNum(Integer.valueOf(classNum));
        return student.toString();
    }
}
```

# 一种更优雅的参数解析方式

这个Controller把请求的参数值解析到了Student类中。这种做法并不是太优雅，它强依赖于`HttpServletRequest` ,需要从`HttpServletRequest`中取得参数来拼出`Student`来。实际上，把请求参数解析到某个类型的工作，不应该让`Controller`来完成，我们有更好的方法来做参数解析，我们期待的`Controller`应该是这样的：

```java
@Controller
public class AddStudentInfoController {

    @RequestMapping("/data/addStudent")
    public String  addStudent(Student student){
        return student.toString();
    }
}
```

我们需要有一个地方（不是在`Controller`），来接收请求的参数，并把参数解析到`Student`中，最后传递到`AddStudentInfoController. addStudent`这里来。要完成这个过程，我们需要实现`HandlerMethodArgumentResolver`接口，来定义一个**参数分解器**。这个接口提供了两个方法：

``` java
//判断给定的MethodParameter类型参数parameter是否被分析器所支持
boolean supportsParameter(MethodParameter parameter);
```

``` java
//将请求中的参数值解析为某种对象。
@NullableObject 
resolveArgument(
MethodParameter parameter,
@Nullable ModelAndViewContainer mavContainer,      		
NativeWebRequest webRequest, 
@Nullable WebDataBinderFactory binderFactory)    throws Exception;
```

我们先来看如何实现上述请求参数解析过程，再来细说这两个方法。我们实现`HandlerMethodArgumentResolver`接口：

```java
public class AddStudentResolver implements HandlerMethodArgumentResolver{

    //判断给定的MethodParameter类型参数parameter是否被分析器所支持
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        //当paramester的参数类型为Student时，支持
        //支持时，会调用resolveArgument函数
        return parameter.getParameterType().equals(Student.class);
    }

    @Nullable
    @Override
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        //这里做具体的解析逻辑
        String name = webRequest.getParameter("name");
        String classNum = webRequest.getParameter("classNum");
        Student student = new Student();
        student.setName(name);
        student.setClassNum(Integer.valueOf(classNum));
        return student;
    }
}

```

除了通过参数的类型（Class<?>）来判断是否要执行参数解析外，也可以用注解的方式来判断是否需要解析。例如，我们定义一个注解，拥有这个注解的参数，将被执行解析：

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ToResolver {
}
```

Controller中的参数，使用`@ToResolver`来注解：

```java
@Controller
public class AddStudentInfoController {

    @RequestMapping("/data/addStudent")
    public String  addStudent(@ToResolver Student student){
        return student.toString();
    }
}
```

然后**参数分解器**的`supportsParameter函数`改成如下：

```java
    //判断给定的MethodParameter类型参数parameter是否被分析器所支持
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        //当paramester 使用@ToResolver注解时，支持
        //支持时，会调用resolveArgument函数
        return parameter.hasParameterAnnotation(ToResolver.class);
    }
```

这时，凡是有`@ToResolver`注解的参数，都会调用对应的`resolveArgument函数`来进行参数解析。

# 深入分析HandlerMethodArgumentResolver

