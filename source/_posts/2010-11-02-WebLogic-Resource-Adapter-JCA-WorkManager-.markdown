---
layout: post
title: "WebLogic在非Resource Adapter里直接使用JCA WorkManager的失败尝试"
comments: true
date: 2010-11-02 05:43
tags:
- SoftwareDev
- WebLogic
- AppServer
---
JCA 1.5 规范定义了WorkManager API，能方便的进行任务的(多线程)并行处理。按照JCA规范的原意，WorkManager是仅提供给Resource Adapter使用的，容器会提供WorkManager实例并通过BootstrapContext注入到adapter中。可是，有时在其他场合，例如EJB、Servlet中，也希望能将请求拆分，使用WorkManager并行处理，但JEE规范中并没有提供这样的机制。

WebLogic提供了CommonJ WorkManager API，可用于任意应用中，但这个API不在JEE规范之中，因此GlassFish是不支持的。而且，使用WebLogic CommonJ WorkManager时需要在ejb-jar.xml中加入一个类型为commonj.work.WorkManager的resource reference，这样同一个ejb包就不能在不同application server间通用了，因此我还是希望能找到一种通用的方法。

SpringFramework的org.springframework.jca.work包为GlassFish和JBoss提供了在Resource Adapter外部使用JCA 1.5 WorkManager的简便方法。看了for GlassFish的源代码，发现它其实是调用了GlassFish内部的factory去直接创建WorkManager，那么对GlassFish是否也可以这样做呢？

反编译WebLogic的jar包，发现commonj WorkManager和connector WorkManager实际上都是delegate到一个weblogic.work.WorkManager实现的，也就是说在WebLogic里，CommonJ和JCA的WorkManager的底层实现其实都是同样的。接下来找到了weblogic.work.WorkManagerFactory，它有find和findOrCreate方法可返回weblogic.work.WorkManager实例，也许就是可以拿到weblogic-ejb-jar.xml里面定义的weblogic WorkManager。拿到了内部的weblogic.work.WorkManager实例后，再调用weblogic.connector.work.WorkManager的create方法，将这个内部WorkManger包裹到JCA WorkManager中，那么我们的应用就可以使用这个JCA WorkManager了，通过Spring的WorkManagerTaskExecutor来很方便的调度任务，上层的逻辑就可以做到与应用服务器无关。看起来很美。

[![WLS-WorkManager](http://farm2.static.flickr.com/1208/5137622842_a9b4e5fda8_b.jpg)](http://www.flickr.com/photos/leoliang/5137622842/)

可是结果是残酷的。写了一个测试程序，结果是通过这个WorkManagerFactory可以成功的获取到weblogic.work.WorkManager实例，也能够创建出包裹的JCA WorkManager，可是真正要使用它来执行work时，它很坚定的拒绝了，抛出异常信息：Attempt to create Work outside the context of the Connector container。

查看代码，发现weblogic.connector.work.WorkManager及WorkImpl内部会寻找相应的RAInstanceManager，并且WorkImpl需要用到rarClassloader，看来它设计上还是与resource adapter相关的。一涉及到classloader，我无心恋战了，算了，就此打住。

P.S. 既然说到WorkManager，就推荐一下我的同事写的相关的一篇blog：[application server 下的任务异步/并行执行方案](http://www.javaeye.com/topic/623825)

P.P.S. 我后来采用的解决方案是让build system可以根据需要为不同的 target application server 选择不同的 deployment descriptor 来build相应的artifacts。

P.P.P.S. 回头再读一下，发现文章里中文英文混杂得很厉害。没办法了，那些专业术语都是看惯英文了，用中文翻译了反而别扭。
