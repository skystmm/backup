---
title: Tomcat源码学习（二）——启动流程之load初探
date: 2018-11-05 20:50:02
tags: [java,tomcat,源码]
categories: java
---
>上次对于Bootstrap类进行了学习，并且引出了Tomcat启动过程中一直有调用的Catalina类，今天就对Catalina类进行学习和分析。

<!-- more -->

根据Bootstrap类的main方法的调用顺序如下图所示：
![Tomcat启动Bootstrap调用顺序](https://upload-images.jianshu.io/upload_images/136194-3d870c4093e696d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



Bootstrap的实例在执行load方法实际调用的是Catalina的load方法。查看Catalina的源码可以发现有两个load方法。
```java
  public void load(String args[]); //对参数进行解析后在调用load()方法
  public void load();
```
load的具体实现是在无参的load方法中，下面来看一下load方法的实现
```java
public void load() {

        if (loaded) {
            return;
        }
        loaded = true;

        long t1 = System.nanoTime();

        initDirs();

        // Before digester - it may be needed
        initNaming();

        // Create and execute our Digester
        Digester digester = createStartDigester();

        InputSource inputSource = null;
        InputStream inputStream = null;
        File file = null;
        try {
            try {
                file = configFile();
                inputStream = new FileInputStream(file);
                inputSource = new InputSource(file.toURI().toURL().toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail", file), e);
                }
            }
            if (inputStream == null) {
                try {
                    inputStream = getClass().getClassLoader()
                        .getResourceAsStream(getConfigFile());
                    inputSource = new InputSource
                        (getClass().getClassLoader()
                         .getResource(getConfigFile()).toString());
                } catch (Exception e) {
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("catalina.configFail",
                                getConfigFile()), e);
                    }
                }
            }

            // This should be included in catalina.jar
            // Alternative: don't bother with xml, just create it manually.
            if (inputStream == null) {
                try {
                    inputStream = getClass().getClassLoader()
                            .getResourceAsStream("server-embed.xml");
                    inputSource = new InputSource
                    (getClass().getClassLoader()
                            .getResource("server-embed.xml").toString());
                } catch (Exception e) {
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("catalina.configFail",
                                "server-embed.xml"), e);
                    }
                }
            }


            if (inputStream == null || inputSource == null) {
                if  (file == null) {
                    log.warn(sm.getString("catalina.configFail",
                            getConfigFile() + "] or [server-embed.xml]"));
                } else {
                    log.warn(sm.getString("catalina.configFail",
                            file.getAbsolutePath()));
                    if (file.exists() && !file.canRead()) {
                        log.warn("Permissions incorrect, read permission is not allowed on the file.");
                    }
                }
                return;
            }

            try {
                inputSource.setByteStream(inputStream);
                digester.push(this);
                digester.parse(inputSource);
            } catch (SAXParseException spe) {
                log.warn("Catalina.start using " + getConfigFile() + ": " +
                        spe.getMessage());
                return;
            } catch (Exception e) {
                log.warn("Catalina.start using " + getConfigFile() + ": " , e);
                return;
            }
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // Stream redirection
        initStreams();

        // Start the new server
        try {
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error("Catalina.start", e);
            }
        }

        long t2 = System.nanoTime();
        if(log.isInfoEnabled()) {
            log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
        }
    }
```
第一眼看到load方法的代码，会觉得这个方法的代码很冗长，详细看一下代码会发现，load方法的主要代码如下：
```java
Digester digester = createStartDigester(); //创建xml的解析规则
file = configFile();//获取配置文件的信息
inputStream = new FileInputStream(file);
inputSource = new InputSource(file.toURI().toURL().toString());
inputSource.setByteStream(inputStream);
digester.push(this);
digester.parse(inputSource);//进行xml的解析
getServer().setCatalina(this);//设置Server的catalina
getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());
initStreams();
getServer().init();
```
这么看下来，load方法其实做了三件事，一件是通过createStartDigester方法创建默认的Tomcat服务配置信息；第二件事是去读取conf/server.xml信息并解析和更新默认的配置信息；最后一件事就是根据合并后的配置信息来对tomcat server进行初始化。上面的load核心代码中，大部分都是在做server.xml的解析准备和解析工作，而真正的Server初始化则是在```getServer().init()```进行的。
在digerster中可以看到，默认使用的server是org.apache.catalina.core.StandardServer，首先我们来看看StandardServer的UML图
![StandardServerUML图](https://upload-images.jianshu.io/upload_images/7504708-2eb08e702eabeb3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中StandardServer继承了LifecycleMBeanBase抽献类，并且实现了Server接口。点进去getServer().init()的方法，会发现init方法是LifecycleBase抽象类中的一个模板方法。
```java
 @Override
    public final synchronized void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }

        try {
            setStateInternal(LifecycleState.INITIALIZING, null, false);
            initInternal();
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }
```
可以看到具体的实现在对应实现类的initInternal方法中，下面来一起看看StandardServer中server的init过程。
```java
@Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");

        // Register the MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");

        // Register the naming resources
        globalNamingResources.init();

        // Populate the extension validator with JARs from common and shared
        // class loaders
        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            // Walk the class loader hierarchy. Stop at the system class loader.
            // This will add the shared (if present) and common class loaders
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File (url.toURI());
                                if (f.isFile() &&
                                        f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException e) {
                                // Ignore
                            } catch (IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();
        }
    }
```
StanderServer的initInternal方法，主要是进行了自己serverice的初始化工作，依次执行每个service的init方法。完成了Server/Service的初始化，这里Catalina的load方法就结束了。通过查看Digster实例的创建代码的和server.xml，可以了解到tomcat默认的server/service为org.apache.catalina.core.StandardService。接下来我们就来看看StandardService的init都做了些什么。

看源码之前，先来看一下StandardService的UML类图
![StandardService类图](https://upload-images.jianshu.io/upload_images/7504708-967d0930242f2402.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到StandardService继承了LifecycleMBeanBase抽象类，实现了Service接口。其中LifecycleMBeanBase抽象类是老朋友了，之前的StandardServer也是继承了这个抽象类，既然StandardService也继承了它，那我们就跳过init这个使用模板设计模式的方法，直接来看看StandardService的初始化都做了些什么。废话不多直接上源码
```java
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (engine != null) {
            engine.init();
        }

        // Initialize any Executors
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // Initialize mapper listener
        mapperListener.init();

        // Initialize our defined Connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.init();
            }
        }
    }
```
看到这段代码，首先会发现其中有很多init方法，然后会发现这些众多的init方法的实例的实现类都继承了LifecycleMBeanBase抽象类。在service的中初始化了container(这里的engine实例),多个连接器（connector）,多个executor和一个mapperListener。

使用默认的server.xml时，container使用的是默认的StandardEngine,作为默认的tomcat容器。connector是根据server.xml中的Connector标签中配置的信息，即对外暴露的端口连接器信息。
```xml
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```

下面来看下一load过程的流程图
  ![load流程图](https://upload-images.jianshu.io/upload_images/7504708-27efe74fc8370abb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此catalina的load过程就完成了。回顾一下load过程首先是是创建了一个digster对象，这个digster对象可以说是对于我们了解load过程，一个很重要的对象，可以帮我们梳理load过程和加载了那些对象，然后读取了server.xml中的内容，server.xml中的内容是对于digster的一个补充的完善；然后根据配置信息初始化Server容器，以及service相关初始化工作。
