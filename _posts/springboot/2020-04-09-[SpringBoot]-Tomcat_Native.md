---
layout: post
title:  "[SpringBoot]SpringBoot - Tomcat APR模式"
date:   2019-7-4 20:00:00 +0800
tags: SpringBoot2.X
color: rgb(26,62,110)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: '关系型数据'
---



# **SpringBoot2.x Tomcat9+ 启动APR报错:校验系统是否安装APR环境**
## APR介绍
> 安装APR后，在处理静态资源的时候速度更快，总而言之就是使用本地的apr库提升处理效率。在没有安装APR情况下，启动tomcat会出现警告信息：The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /Users/username/Library/Java/Extensions…

### 1.安装需要
{% highlight text %} 
    1.apr下载地址：http://www.apache.org/dist/apr
    2.apr-util下载地址：http://www.apache.org/dist/apr
    3.OpenSSL命令安装： brew install openssl
    4.gcc命令安装：brew install gcc
    5.tomcat-native下载地址：http://tomcat.apache.org/native-doc/
{% endhighlight %}


### 2.安装apr

#### **解压下载的安装包**
{% highlight shell %} 
    cd apr-1.x.x 
    ./configure  
    make  
    make install  
    apr 默认安装在 /usr/local/apr
{% endhighlight %}

### 3.安装apr-util

#### **解压下载的安装包**
{% highlight shell %} 
    cd apr-util-1.3.12  
    ./configure --with-apr=/usr/local/apr  
    make  
    make install 
{% endhighlight %}
 
### 4.安装 tomcat-native 

#### **解压下载的安装包**
{% highlight shell %} 
    cd tomcat-native/jni/native  
    sudo ./configure --with-apr=/usr/local/apr/bin/apr-1-config --with-java-home=$JAVA_HOME --prefix=$CATALINA_HOME --with-ssl=/usr/local/opt/openssl
    # java-home的路径就是JAVA_HOME的路径 
    sudo make  
    sudo make install  
{% endhighlight %}

### 5.设置 apr 的环境变量： 
{% highlight shell %} 
    vi ~/.bash_profile
    # 后面添加以下内容  
    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib  
    # 使profile生效，  
    source ~/.bash_profile
    
    如果成功，在目录/usr/local/apr/lib/下会生成一个名为libtcnative-1.0.dylib的库文件，使用ln命令做一个软链接到上述警告信息中提到的一个目录即可，例如：
    ln -s /usr/local/apr/lib/libtcnative-1.dylib /Library/Java/Extensions/
{% endhighlight %}






# **SpringBoot2.x Tomcat9+ 整合异常**
{% highlight java %} 
    2020-04-09 18:16:03.104 ERROR 67421 --- [           main] o.a.catalina.core.AprLifecycleListener   : Failed to initialize the SSLEngine.
    
    org.apache.tomcat.jni.Error: 70023: This function has not been implemented on this platform
    	at org.apache.tomcat.jni.SSL.initialize(Native Method) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
    	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:na]
    	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.__invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
    	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:45009) ~[na:na]
    	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:45012) ~[na:na]
    	at java.base/java.lang.reflect.Method.invoke(Method.java:567) ~[na:na]
    	at org.apache.catalina.core.AprLifecycleListener.initializeSSL(AprLifecycleListener.java:289) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.AprLifecycleListener.lifecycleEvent(AprLifecycleListener.java:136) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.fireLifecycleEvent(LifecycleBase.java:123) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.setStateInternal(LifecycleBase.java:423) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:135) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:173) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
    	at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at java.base/java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:140) ~[na:na]
    	at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.StandardHost.startInternal(StandardHost.java:841) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
    	at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at java.base/java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:140) ~[na:na]
    	at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.StandardEngine.startInternal(StandardEngine.java:262) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.StandardService.startInternal(StandardService.java:421) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.core.StandardServer.startInternal(StandardServer.java:930) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.apache.catalina.startup.Tomcat.start(Tomcat.java:468) ~[tomcat-embed-core-9.0.33.jar:9.0.33]
    	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.initialize(TomcatWebServer.java:107) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.<init>(TomcatWebServer.java:88) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getTomcatWebServer(TomcatServletWebServerFactory.java:438) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getWebServer(TomcatServletWebServerFactory.java:191) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:180) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:153) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.context.support.AbstractApplicationContext.__refresh(AbstractApplicationContext.java:544) ~[spring-context-5.2.5.RELEASE.jar:5.2.5.RELEASE]
    	at org.springframework.context.support.AbstractApplicationContext.jrLockAndRefresh(AbstractApplicationContext.java:40002) ~[spring-context-5.2.5.RELEASE.jar:5.2.5.RELEASE]
    	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:41008) ~[spring-context-5.2.5.RELEASE.jar:5.2.5.RELEASE]
    	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:141) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:747) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.SpringApplication.run(SpringApplication.java:315) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1215) ~[spring-boot-2.2.6.RELEASE.jar:2.2.6.RELEASE]
    	at com.microboot.cn.MicroBootMybatisApplication.main(MicroBootMybatisApplication.java:20) ~[classes/:na]
{% endhighlight %}

### 上面异常需要开启：
#### Tomcat配置: 
##### 1.TomcatConfiguration.java
{% highlight java %} 
package com.microboot.cn.configuration;

import org.apache.catalina.connector.Connector;
import org.apache.catalina.core.AprLifecycleListener;
import org.apache.coyote.http11.Http11AprProtocol;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TomcatConfiguration {

    @Value("${server.tomcat.apr.enabled:false}")
    private boolean enabled;


    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        // 设置协议
        factory.setProtocol("org.apache.coyote.http11.Http11AprProtocol");
        // factory.setProtocol("org.apache.coyote.http11.Http11Nio2Protocol");
        if (enabled) {
            // 开启APR
            AprLifecycleListener  aprLifecycleListener = new AprLifecycleListener();
            aprLifecycleListener.setSSLEngine("off");
            factory.addContextLifecycleListeners(aprLifecycleListener);
        }

        factory.addConnectorCustomizers(new TomcatConnectorCustomizer() {
            @Override
            public void customize(Connector connector) {
                Http11AprProtocol  handler = (Http11AprProtocol)connector.getProtocolHandler();
                //对tomcat进行对应的设置
                boolean isAprRequired = handler.isAprRequired();
                System.out.println("isAprRequired:"+isAprRequired);


            }
        });
        return factory;
    }
}

{% endhighlight %}

##### 2.application.properties
{% highlight properties %} 
######################################## Tomcat请求模式 SATRT #############################################
######### 1.BIO
######### 2.NIO
######### 3.APR
######################################## Tomcat请求模式 END #############################################
##### 开启APR
server.tomcat.apr.enabled=false

{% endhighlight %}

# **注意：SpringBoot2.x支持的JDK最低版本版本**