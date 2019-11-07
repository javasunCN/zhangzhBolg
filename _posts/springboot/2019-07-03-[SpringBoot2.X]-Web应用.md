---
layout: post
title:  "[SpringBoot]SpringBoot2.0 - Web应用(一)"
date:   2019-7-3 20:00:00 +0800
tags: SpringBoot2.X
color: rgb(26,62,110)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Web应用介绍'
---



# **传统Servlet应用**
{% highlight yml %} 
    ✅ Servlet组件: Servlet、Filter、Listener
    ✅ Servlet注册: Servlet注解、Spring Bean、 RegistrationBean
    ✅ 异步非阻塞: 异步Servlet3.0、非阻塞Servlet3.1
{% endhighlight %}
---
# **Servlet应用 - 实践**
### 依赖 - pom.xml
{% highlight xml %} 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
{% endhighlight %}


### Servlet

* 实现
    * @WebServlet
    * HttpServlet
* URL映射
    * @WebServlet(urlPatterns = "/my/servlet")
* 注册
    * @ServletComponentScan(basePackages = {"om.imoc.imoc.web.servlet"})

{% highlight java %} 
    @WebServlet(urlPatterns = "/my/servlet")
    public class DefaultServlet extends HttpServlet {
    
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            resp.getWriter().println("自定义Servlet");
        }
    }       
{% endhighlight %}
{% highlight java %}    
@SpringBootApplication
@ServletComponentScan(basePackages = {"om.imoc.imoc.web.servlet"})
public class ImocApplication {

	public static void main(String[] args) {
		SpringApplication.run(ImocApplication.class, args);
	}

}
         
{% endhighlight %}



### SprinngBean

- 实现
- URL映射

{% highlight yml %} 
         
{% endhighlight %}




### RegistrationBean

- 实现
- URL映射

{% highlight yml %} 
         
{% endhighlight %}

---
# **异步非阻塞**
### 异步Servlet
{% highlight yml %} 
         * javax.servlet.ServletRequest#startAsync()
         * javax.servlet.AsyncContext
{% endhighlight %}
```java
@WebServlet(urlPatterns = "/my/servlet",
        asyncSupported = true
    )
public class DefaultServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 异步Servlet,@WebServlet(
        //         开启异步调用
        //        asyncSupported = true
        //    )
        AsyncContext context = req.startAsync();

        context.start(() -> {
            try {
                resp.getWriter().println("中文乱码Servlet");
                // 触发完成，不然一致等待返回，结果超时
                context.complete();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }


}

```
```java
@SpringBootApplication
@ServletComponentScan(basePackages = {"com.imoc.imoc.web.servlet"})
public class ImocApplication {

	public static void main(String[] args) {
		SpringApplication.run(ImocApplication.class, args);
	}

}
```

### 非阻塞Servlet
{% highlight yml %} 
        * javax.servlet.ServletInputStream#setReadListener
            * javax.servlet.ReadListener
        * javax.servlet.ServletOutputStream#setWriteListener
            * javax.servlet.WriteListener
{% endhighlight %}

---
# **Spring Web MVC 应用**
{% highlight text %} 
        * Web MVC视图: 模板引擎、内容协商、异常处理等
        * Web MVC REST: 资源服务、资源跨域、服务发现
        * Web MVC核心: 核心架构、处理流程、核心组件
{% endhighlight %}

### Web MVC视图
{% highlight text %} 
        * ViewResolver
        * View
{% endhighlight %}

### 模板引擎
{% highlight text %} 
        * Thymeleaf
        * Freemarker
        * JSP
{% endhighlight %}

### 内容协商 - 帮助选择合适的模板引擎进行处理
{% highlight text %} 
        * ContentNegotiationConfigurer
        * ContentNegotiationStrategy
        * ContentNegotiatingViewResolver
{% endhighlight %}


### 异常处理
{% highlight text %} 
        * @ExceptionHandler
        * HandlerExceptionResolver
            * ExceptionHandlerExceptionResolver
        * BasicErrorController(Spring Boot)
{% endhighlight %}

---
# **Web MVC REST**
### 资源服务
{% highlight text %} 
        * @RequestMapping
            * @GetMapping
            * @PostMapping
            * @PutMapping
            * @DeleteMapping
        * @ResponseBody
        * @RequestBody
{% endhighlight %}

### 资源跨域
{% highlight text %} 
        * 传统解决方案:
            * ifreame
            * JSONP
        * CrossOrigin (Spring FreamWork)
        * org.springframework.web.servlet.config.annotation.WebMvcConfigurer#addCorsMappings (Spring FreamWork)
{% endhighlight %}

### 服务发现
{% highlight text %} 
        * HATEOS
{% endhighlight %}

---
# **Web MVC 核心**
### 核心架构
{% highlight text %} 
        
{% endhighlight %}

### 处理流程
{% highlight text %} 
        
{% endhighlight %}

### 核心组件
{% highlight text %} 
        * DispatcherServlet
        * HandlerMapping
        * HandlerAdapter
        * ViewResolver
        * ...
{% endhighlight %}

---
# **Spring Web Flux应用**

- 对Serlvlet的补充

### Reactor基础
{% highlight text %} 
        * Java Lambda
        * Mono (核心接口)
        * Flux (核心接口)
{% endhighlight %}

### WebFlux核心
{% highlight text %} 
        * WebMVC注解
        * 函数式声明 
            * RouterFunction
        * 异步非阻塞
{% endhighlight %}

### WebFlux优势和限制
{% highlight text %} 
        * 页面渲染
        * REST应用 
{% endhighlight %}


--- 
# **Web Server应用**
## 切换Web Server
### Tomcat -> Jetty
{% highlight xml %} 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 排除默认的Tomcat -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 添加jetty依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
        
{% endhighlight %}

### Tomcat -> undertow
{% highlight xml %} 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 排除默认的Tomcat -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 添加undertow依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
        
{% endhighlight %}

### 传统Servlet容器 -> Webflux
{% highlight xml %} 
<!-- 注释掉 spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
        
{% endhighlight %}

## 自定义Servlet Web Server
{% highlight text %} 
    * WebServerFactoryCustomizer
        
{% endhighlight %}

## 自定义Reactive Web Server
{% highlight text %} 
    * ReactiveWebServerFactoryCustomizer
        
{% endhighlight %}





