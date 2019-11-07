---
layout: post
title:  "[SpringBoot]SpringBoot2.0 - 1"
date:   2019-7-2 20:00:00 +0800
tags: SpringBoot2.X
color: rgb(26,62,110)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: '为什么说SpringBoot易学难精?'
---



# **SpringBoot易学**
{% highlight yml %} 
    ✅ 组件自动装配: 规约大于配置, 专注核心业务
    ✅ 外部化配置: 一次构建、按需调配、导出运行
    ✅ 嵌入式容器: 内置容器、无需部署、独立运行
    ✅ Spring Boot Starter: 简化依赖、按需装配、自我包含
    ✅ Production-Ready: 一站式运维、生态无缝整合
    
{% endhighlight %}

# **SpringBoot难精**
{% highlight yml %} 
   ❎ 组件自动装配: 模式注解、@Enable模块、条件装配、加载机制
   ❎ 外部化配置: Environment抽象、声明周期、破坏性变更(Springboot1.x与Springboot2.x部分不兼容)
   ❎ 嵌入式容器: Servlet Web容器、Reactive Web容器
   ❎ Spring Boot Starter: 依赖管理、装配条件、装配顺序
   ❎ Production-Ready: 健康检查、数据指标、@Endpoint管控
   
    
{% endhighlight %}

# **SpringBoot与JavaEE规范**
{% highlight yml %} 
   🆕 Web: Servlet(JSR-315\JSR-340)
   🆕 SQL: JDBC(JSR-221)
   🆕 数据校验: Bean Validation(JSR-303 \ JSR-349)
   🆕 缓存: Java Caching API (JSR-107)
   🆕 WebSockets: Java API For WebSocket(JSR-356)
   🆕 Web Services: JAX-WS (JSR-224)
   🆕 Java管理: JMX(JSR3)
   🆕 消息: JMS(JSR-914)
{% endhighlight %}

# **SpringBoot核心特性**
{% highlight yml %} 
   🏷 组件自动装配: WebMvc、WebFlux、JDBC等
        ※ 激活: @EnableAutoConfiguration
        ※ 配置: /META-INF/spring.factories
        ※ 实现: XXXAutoConfiguration
   🏷 嵌入式web容器: Tomcat、Jetty 以及Undertow 
        ※ WebServlet: Tomcat、Jetty 以及Undertow 
        ※ WebReactive: Netty Web Server
   🏷 生产准备特性: 指标、健康检查、外部化配置等
        ※ 指标: /actuator/metrics
        ※ 健康检查: /actuator/health
        ※ 外部化配置:/actuator/configprops
{% endhighlight %}


