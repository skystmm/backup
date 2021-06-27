---
title: Tomcat源码学习（一）——Bootstarp入口类
date: 2018-11-05 20:40:55
tags: [java,tomcat,源码]
categories: java
---

Tomcat作为平时工作中出镜率最高的web容器，今天我们就来对其源码一探究竟。通过github获取了tomcat的最新代码，获取地址如下
>https://github.com/apache/tomcat

这里使用idea的git直接从github上拉去了最新源码，同时方便获取最新修改。

<!-- more -->

获得源码后，首先来探索一下Tomcat的启动过程，那就要来看看他的启动入口类org.apache.catalina.startup.BootStrap。
找到了入口类，从main方法开始入手，代码如下：
```java
public static void main(String args[]) {
        // 同步初始化Bootstrap对象,不太理解这里为什么要加同步锁
        synchronized (daemonLock) {
            if (daemon == null) {
                // Don't set daemon until init() has completed
                Bootstrap bootstrap = new Bootstrap();
                try {
                    bootstrap.init();
                } catch (Throwable t) {
                    handleThrowable(t);
                    t.printStackTrace();
                    return;
                }
                daemon = bootstrap;
            } else {
                // When running as a service the call to stop will be on a new
                // thread so make sure the correct class loader is used to
                // prevent a range of class not found exceptions.
                Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
            }
        }

        try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            } else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }

    }
```
上述是tomcat的main方法，如果我只看tomcat的start过程，代码可以简化为：
```java
   if (daemon == null) {
    Bootstrap bootstrap = new Bootstrap();
    bootstrap.init();
    daemon = bootstrap;
  }
  String command = "start";
  if (command.equals("start")) {
    daemon.setAwait(true);
    daemon.load(args);
    daemon.start();
    if (null == daemon.getServer()) {
      System.exit(1);
    }
  }
```
  从上面代码就可以清晰的看出来tomcat 启动流程，首先创建bootstrap对象，并执行init方法初始化bootstrap,初始化完成后将新创建的bootstrap实例赋值给daemon对象，在启动时，daemon会先调用Bootstrap的load方法和start方法来完成Tomcat的启动工作。

从main方法中可以发现启动过程中主要的三个方法init,load和start。依次来看一下这三个方法的实现，来了解一下tomcat启动具体做了些什么。
```java
public void init() throws Exception {

        initClassLoaders(); //类加载器初始化

        Thread.currentThread().setContextClassLoader(catalinaLoader);

        SecurityClassLoad.securityClassLoad(catalinaLoader);


        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.getConstructor().newInstance();

        // Set the shared extensions class loader
        if (log.isDebugEnabled())
            log.debug("Setting startup class properties");
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;

    }
```
可以看到init方法是设置了类加载器，同时通过类加载器，生成了一个Catalina对象实例，同时设置了catalina对象的父类加载器。并将生成的catalina对象赋值给了bootstrap对象的catalinaDaemon属性。上述操作，总的来看init方法做了一件事情，就是给bootstarp的catalinaDanmon属性进行了初始化。
再来看看load方法做了什么事情，核心代码如下：
```java

        Method method =
            catalinaDaemon.getClass().getMethod(methodName, paramTypes);
        if (log.isDebugEnabled())
            log.debug("Calling startup class " + method);
        method.invoke(catalinaDaemon, param);


```
可以看到load方法其实是通过反射的方法调动了catalina的load方法。进行基础信息的加载。catalina的load将在后续文章中详细介绍。
```java
  if( catalinaDaemon==null ) init();

        Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
        method.invoke(catalinaDaemon, (Object [])null);
```
start方法也很简单，依然是通过反射的方法调用了catalina的start 方法。

翻看了其他的Bootstarp类方法，大部分都是通过反射去调用了对应的Catalina类的对应方法来完成对应操作。

可以看到Bootstrap类如其名，作为tomcat的入口类，同时起到了一个引导的作用，通过阅读和分析，也引出了tomcat启动过程中一个重要的类Catalina。

>对于源码中同步疑问，在万能的stackoverflow提问后，理解了为什么要对daemon的初始化过程进行同步操作
[传送门](https://stackoverflow.com/questions/52980662/why-tomcat-lastest-code-use-synchronized-in-main-method)
