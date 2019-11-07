---
layout: post
title:  "Java Core之从字节码深入理解HelloWorld"
date:   2019-10-01 18:30:00 +0800
tags: JavaCore
color: rgb(255,210,32)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/cxy.jpg?raw=true'
subtitle: 'Java Core之从字节码深入理解HelloWorld'
---


# 1 JavaCore学习思想
> 思想核心:点连成线、线连城面、面组成体积

{% highlight text %}
    JAVA中的点: Java基础语法
    JAVA中的线: Java核心技术
    JAVA中的面: Java框架:Netty/Spring/Mybatis等
    JAVA中的体积: 实践应用或者项目
    
{% endhighlight %} 

# HelloWorld入门
```java
// [权限修饰符] class 类名
public class Hello {
    // JVM规定程序入口方法: main
    // JVM将字节码文件读入到内存中，会找到包含main方法的类，执行main方法
	public static void main(String[] args) {
		System.out.println("Hello");
	}
}
```

{% highlight text %}
    1、将.java文件编译成.class文件
    javac HelloWorld.java
    2、运行class文件:校验并装载类到内存中,并执行指令，输出结果
    java HelloWorld
    查看class文件字节码:
    javap -c HelloWorld
{% endhighlight %} 

# 字节码与类创建过程
{% highlight text %}
    类加载器加载字节码文件:
        1、加载
        2、验证
        3、准备
        4、解析
        5、初始化
        6、使用
        7、卸载
    
    在内存中创建对象的执行顺序:
        1、加载实例信息进入开辟的内存中
        2、执行构造方法就是<init>方法
        
    对堆内存中开辟的结构解析:
        1、对象由对象的头部信息和实例信息组成
            头部信息:
                1、对齐填充
                2、持有指向方法区的指针
                3、描述信息(持有当前对象锁的线程ID、持有对象锁的线程个数、在GC中存活的生命周期数、偏向锁标志)
                    偏向锁:HotSpot[1]的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，
                          为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，
                          会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，
                          只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。
                          如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：
                          如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。
                          
                          始终只有一个线程在执行同步块，在它没有执行完释放锁之前，没有其它线程去执行同步块，在锁无竞争的情况下使用，
                          一旦有了竞争就升级为轻量级锁，升级为轻量级锁的时候需要撤销偏向锁，撤销偏向锁的时候会导致Stop The Word操作；
                          
                          高并发的应用会禁用掉偏向锁。
                          
                    轻量级锁:如果持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换
                            进入阻塞挂起状态，它们只需要等一等（自旋），等持有锁的线程释放锁后即可立即获取锁，
                            这样就避免用户线程和内核的切换的消耗。
{% endhighlight %} 




