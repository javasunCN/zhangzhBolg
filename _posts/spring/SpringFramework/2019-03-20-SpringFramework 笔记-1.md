---
layout: post
title:  "Spring源码阅读[一]"
date:   2019-3-20 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[一]'
---

{% highlight text %}
    注：基于Spring版本：4.3.x
{% endhighlight %} 


<center><b><h3>Spring源码阅读 - HelloWord开始</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>项目搭建</h3></b></center>

#### Maven pom.xml
{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.spring4-source.cn</groupId>
    <artifactId>spring4_source</artifactId>
    <version>1.0-SNAPSHOT</version>


    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>

        <springframework.version>4.3.22.RELEASE</springframework.version>

        <commons-logging.version>1.2</commons-logging.version>
    </properties>


    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-beans -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>${springframework.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${springframework.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${springframework.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>${springframework.version}</version>
        </dependency>

        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>${commons-logging.version}</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

    </dependencies>


    <build>
        <plugins>

            <!-- 生成sources源码包的插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.0.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- 生成javadoc文档包的插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <aggregate>true</aggregate>
                </configuration>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>


            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.1</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                    </includes>
                    <excludes>
                        <exclude>**/*NoRunTest.java</exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
        <sourceDirectory>src/main/java</sourceDirectory>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>**/*</include>
                </includes>
                <excludes>
                    <exclude>**/.svn/</exclude>
                </excludes>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
                <includes>
                    <include>**/*</include>
                </includes>
                <excludes>
                    <exclude>**/.svn/</exclude>
                </excludes>
            </testResource>
        </testResources>
    </build>



</project>

{% endhighlight %} 
    
#### Maven修改源码注释
{% highlight text %}
    1、下载Jar的源码 例如:*-sources.jar
    2、解压*-sources.jar到指定的文件夹
    3、Maven中的jar包sources删除，重新指定解压后的文件夹位置，导入本地源文件
{% endhighlight %} 

---------------------------------------

#### 测试代码

> HelloWorld.java

{% highlight java %}
    public class HelloWorld {
    
        private String name;
        public void setName(String name) {
            System.out.println("调用了设置属性");
            this.name = name;
        }
        public HelloWorld(){
            System.out.println("初始化构造器");
        }
        public void hello(){
            System.out.println("Hello: " + name);
        }
    }
{% endhighlight %} 

> applicationContext.xml(Bean注册)

{% highlight xml %}
    <?xml version="1.0" encoding="UTF-8"?>
    
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    
            <bean id="helloWorld" class="com.source.day1.HelloWorld">
                <property name="name" value="测试" />
            </bean>
    </beans>
{% endhighlight %} 

> ApplicationMain.java

{% highlight java %}
    public class ApplicationMain {
    
        public static void main(String[] args) {
            //1、创建Spring的IOC容器的对象
            ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
            //2、从IOC的容器中获取Bean的实例
            HelloWorld helloWorld = (HelloWorld) ctx.getBean("helloWorld");
            //3、调用hello方法
            helloWorld.hello();
        }
    
    }
{% endhighlight %} 






