---
title: 笔记--注解ControllerAdvice的使用
date: 2020-06-29 10:47:53
tags:
- Spring
- 笔记
categories: Spring 
keywords: Spring,@ControllerAdvice,@ExceptionHandler,全局异常处理
---
最近改造一个项目，将SpringMVC升级为SpringBoot，发现这个项目没有做全局异常处理（实际上我手里接过来的好多项目都没做），于是想给加一个全局异常处理的逻辑。通常的做法我比较熟悉的立马能想到的就是利用Spring的异常处理接口HandlerExceptionResolver自定义自己的异常处理器，或者利用AOP的环绕通知来做。这里就不多做介绍了。然后我看到一个比较新的项目中的做法--增强的Controller中使用@ExceptionHandler，以前没有见人用过，特此学习记录一下。

### @ControllerAdvice
这是一个很有用的注解，顾名思义，这是一个增强的Controller，使用这个Controller，可以实现三个方面的功能
* 全局异常处理
* 全局数据绑定
* 全局数据预处理

灵活使用这三个功能，可以帮助我们简化很多工作，需要注意的是，这是SpringMVC提供的功能，在Spring Boot中可以直接使用，下面分别来看。
#### 全局异常处理
使用 @ControllerAdvice 实现全局异常处理，只需要定义类，添加该注解即可定义方式如下：

```java
@ControllerAdvice
public class MyGlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ModelAndView customException(Exception e) {
        ModelAndView mv = new ModelAndView();
        mv.addObject("message", e.getMessage());
        mv.setViewName("myerror");
        return mv;
    }
}
```

在该类中，可以定义多个方法，不同的方法处理不同的异常，例如专门处理空指针的方法、专门处理数组越界的方法...，也可以直接向上面代码一样，在一个方法中处理所有的异常信息。@ExceptionHandler 注解用来指明异常的处理类型，即如果这里指定为 NullpointerException，则数组越界异常就不会进到这个方法中来。

#### 全局数据绑定
全局数据绑定功能可以用来做一些初始化的数据操作，我们可以将一些公共的数据定义在添加了 @ControllerAdvice 注解的类中，这样，在每一个 Controller 的接口中，就都能够访问导致这些数据。

使用步骤，首先定义全局数据，如下：

```java
@ControllerAdvice
public class MyGlobalExceptionHandler {
    @ModelAttribute(name = "md")
    public Map<String,Object> mydata() {
        HashMap<String, Object> map = new HashMap<>();
        map.put("age", 99);
        map.put("gender", "男");
        return map;
    }
}
```

使用 @ModelAttribute 注解标记该方法的返回数据是一个全局数据，默认情况下，这个全局数据的 key 就是返回的变量名，value 就是方法返回值，当然开发者可以通过 @ModelAttribute 注解的 name 属性去重新指定 key。

定义完成后，在任何一个Controller 的接口中，都可以获取到这里定义的数据：
```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(Model model) {
        Map<String, Object> map = model.asMap();
        System.out.println(map);
        int i = 1 / 0;
        return "hello controller advice";
    }
}
```

#### 全局数据预处理
假设有两个实体类，Book 和 Author，分别定义如下：
```java
public class Book {
    private String name;
    private Long price;
    //getter/setter
}
public class Author {
    private String name;
    private Integer age;
    //getter/setter
}
```
此时，如果我定义一个数据添加接口，如下：
```java
@PostMapping("/book")
public void addBook(Book book, Author author) {
    System.out.println(book);
    System.out.println(author);
}
```
这个时候，添加操作就会有问题，因为两个实体类都有一个 name 属性，从前端传递时无法区分。此时通过 @ControllerAdvice 的全局数据预处理可以解决这个问题。

解决步骤如下:
1. 给接口中的变量取别名
```java
@PostMapping("/book")
public void addBook(@ModelAttribute("b") Book book, @ModelAttribute("a") Author author) {
    System.out.println(book);
    System.out.println(author);
}
```

2. 进行请求数据预处理
在 @ControllerAdvice 标记的类中添加如下代码:
```java
@InitBinder("b")
public void b(WebDataBinder binder) {
    binder.setFieldDefaultPrefix("b.");
}
@InitBinder("a")
public void a(WebDataBinder binder) {
    binder.setFieldDefaultPrefix("a.");
}
```
@InitBinder("b") 注解表示该方法用来处理和Book和相关的参数，在方法中，给参数添加一个 b 前缀,即请求参数要有b前缀。

3. 发送请求
请求发送时，通过给不同对象的参数添加不同的前缀，可以实现参数的区分：
```shell
curl -X POST http://127.0.0.1:8888/book?a.name=AAA&a.age=18&b.name=BBB&b.price=99
```