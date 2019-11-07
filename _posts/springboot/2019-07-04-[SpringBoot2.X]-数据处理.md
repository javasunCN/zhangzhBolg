---
layout: post
title:  "[SpringBoot]SpringBoot2.0 - 数据处理(一)"
date:   2019-7-4 20:00:00 +0800
tags: SpringBoot2.X
color: rgb(26,62,110)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: '关系型数据'
---



# **关系型数据**
{% highlight yml %} 
    * JDBC:数据源、JdbcTemplate、自动装配
    * JPA: 实体映射关系、实体操作、自动装配
    * 事务: Spring事务抽象、JDBC事务处理、自动装配
{% endhighlight %}
---
## **JDBC**
### 依赖
{% highlight xml %} 
    <!-- JDBC依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
{% endhighlight %}

### 数据源
{% highlight text %} 
    * javax.sql.DataSource (JDK)
    * org.springframework.jdbc.core.JdbcTemplate
    * 2.x之后添加了新的数据库连接池HikariCP
{% endhighlight %}

### 自动装配
{% highlight text %} 
    * 
{% endhighlight %}







