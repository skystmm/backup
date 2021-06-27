---
title: Tomcat源码学习(三)——启动流程之start
date: 2018-11-11 14:51:03
tags: [java,tomcat,源码]
categories: java
---

> 上一篇对于load过程进行学习，从Bootstrap的入口类可以看到启动过程主要分为三个流程——init,load和start。之前已经学习了init和load的源码，今天就来一探tomcat启动中最后一个流程start
<!-- more -->
![Tomcat启动Bootstrap调用顺序](https://upload-images.jianshu.io/upload_images/136194-3d870c4093e696d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



之前我们看到bootstarp的start方法实际是通过反射调用了Catalina的start方法，接下来就看一下start方法的内容
```java
 /**
     * Start a new server instance.
     */
    public void start() {
        if (getServer() == null) {
            load();
        }
        if (getServer() == null) {
            log.fatal("Cannot start server. Server instance is not configured.");
            return;
        }
        long t1 = System.nanoTime();
        // Start the new server
        try {
            getServer().start();
        } catch (LifecycleException e) {
            log.fatal(sm.getString("catalina.serverStartFail"), e);
            try {
                getServer().destroy();
            } catch (LifecycleException e1) {
                log.debug("destroy() failed for failed Server ", e1);
            }
            return;
        }
        long t2 = System.nanoTime();
        if(log.isInfoEnabled()) {
            log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
        }
        // Register shutdown hook
        if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);

            // If JULI is being used, disable JULI's shutdown hook since
            // shutdown hooks run in parallel and log messages may be lost
            // if JULI's hook completes before the CatalinaShutdownHook()
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                        false);
            }
        }
        if (await) {
            await(); //监听关闭服务连接信息端口并阻塞，默认8005
            stop();
        }
    }
```
首先可以看到，会对server进行npl检测，如果server没有初始化就会调用load方法来进行初始化工作，然后映入眼帘的就是Start方法——今天的正主了；完成start后，会向JVM中添加钩子方法，这里的钩子方法时为了优雅的关闭tomcat。

点到getServer().start()方法，会发现有一次来到了LifecycleBase类，这会不一样的是我们看到的是一个新的模板方法start。让我们来先看看start方法都做了什么
```java
@Override
    public final synchronized void start() throws LifecycleException {

        if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
                LifecycleState.STARTED.equals(state)) {

            if (log.isDebugEnabled()) {
                Exception e = new LifecycleException();
                log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
            } else if (log.isInfoEnabled()) {
                log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
            }

            return;
        }

        if (state.equals(LifecycleState.NEW)) {
            init();
        } else if (state.equals(LifecycleState.FAILED)) {
            stop();
        } else if (!state.equals(LifecycleState.INITIALIZED) &&
                !state.equals(LifecycleState.STOPPED)) {
            invalidTransition(Lifecycle.BEFORE_START_EVENT);
        }

        try {
            setStateInternal(LifecycleState.STARTING_PREP, null, false);
            startInternal();
            if (state.equals(LifecycleState.FAILED)) {
                // This is a 'controlled' failure. The component put itself into the
                // FAILED state so call stop() to complete the clean-up.
                stop();
            } else if (!state.equals(LifecycleState.STARTING)) {
                // Shouldn't be necessary but acts as a check that sub-classes are
                // doing what they are supposed to.
                invalidTransition(Lifecycle.AFTER_START_EVENT);
            } else {
                setStateInternal(LifecycleState.STARTED, null, false);
            }
        } catch (Throwable t) {
            // This is an 'uncontrolled' failure so put the component into the
            // FAILED state and throw an exception.
            handleSubClassException(t, "lifecycleBase.startFail", toString());
        }
    }
```
看到start的实现，会发现比init方法处理复杂了很多，细看会发现start方法会对当前的LifecycleBase实例的状态进行判断，然后调用startInternal方法，然后根据startInternal的结果，再次更新该实例的状态。相当于一个流程控制。这里又引出了startInternal方法，这个就会在继承了LifecycleBase的实现类中实现了，上文中提到过tomcat默认的server是StandardServer，那么就来看看server的Start到底做了些什么。
```java
 @Override
    protected void startInternal() throws LifecycleException {

        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);

        globalNamingResources.start();

        // Start our defined Services
        synchronized (servicesLock) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();
            }
        }
    }
```
从代码汇总可以看到，StandardServr的start过程是先根据Server实例创建生命周期事件实例，再将该事件实例依次注册到配置的listener上。最后一次执行Service的start方法来启动配置的ServerService。在对load过程学习的时候，我们知道默认情况下ServerServIce只有一个——StandardService。
```java
 @Override
    protected void startInternal() throws LifecycleException {

        if(log.isInfoEnabled())
            log.info(sm.getString("standardService.start.name", this.name));
        setState(LifecycleState.STARTING);

        // Start our defined Container first
        if (engine != null) {
            synchronized (engine) {
                engine.start();
            }
        }

        synchronized (executors) {
            for (Executor executor: executors) {
                executor.start();
            }
        }

        mapperListener.start();

        // Start our defined Connectors second
        synchronized (connectorsLock) {
            for (Connector connector: connectors) {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            }
        }
    }
```
可以看到start过程和load过程是对应的，不过相对于load过程，对于container，executor和connector的start都使用了同步操作。

那么就先来看看Container的启动过程，根据load学习，得知了默认的container是StandardEngine，它的UML关系如下
![StandardEngine 继承关系](https://upload-images.jianshu.io/upload_images/7504708-cd41e2e41f6b61bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

阅读源码会发现StandardEngine的启动过程，是在其父类ContainerBase中实现的，话不多说先来一波源码
```java
 @Override
    protected synchronized void startInternal() throws LifecycleException {
        // Start our subordinate components, if any
        logger = null;
        getLogger();
        Cluster cluster = getClusterInternal();
        if (cluster instanceof Lifecycle) {
            ((Lifecycle) cluster).start();
        }
        Realm realm = getRealmInternal();
        if (realm instanceof Lifecycle) {
            ((Lifecycle) realm).start();
        }
        // Start our child containers, if any
        Container children[] = findChildren();
        List<Future<Void>> results = new ArrayList<>();
        for (int i = 0; i < children.length; i++) {
            results.add(startStopExecutor.submit(new StartChild(children[i])));
        }
        MultiThrowable multiThrowable = null;
        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Throwable e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                if (multiThrowable == null) {
                    multiThrowable = new MultiThrowable();
                }
                multiThrowable.add(e);
            }
        }
        if (multiThrowable != null) {
            throw new LifecycleException(sm.getString("containerBase.threadedStartFailed"),
                    multiThrowable.getThrowable());
        }
        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).start();
        }
        setState(LifecycleState.STARTING);
        // Start our thread
        threadStart();
    }
```
在container的启动过程中，会异步的对container的子container进行Start的操作。并通过Future来获取其子container的Start结果，如果其中任何一个子container启动异常，都会添加到multiThrowable实例中，然后抛出异常。最后拉起Container的校验session有效期的线程,同时该线程也是一个守护线程。同时在启动Container的时，会对webapp(在server.xml中可进行配置)目录下，自己发布的应用程序进行加载和初始化。

接下来对配置的工作线程池进行启动操作，源码中提供的默认server.xml中并没有配置工作线程池，不配置时所有的connector共享一个默认线程池，配置后connector可以使用专属的线程池,给8080端口配置使用名称为tomcatThreadPool的线程池，方法如下
```xml
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>
<Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```
访问8080端口时，将会使用tomcatThraeadPool这个线程池中的线程进行处理，而不是默认线程池。最后进行connector的启动。到这里tomcat完成了整个启动过程，可以对外提供服务了。
