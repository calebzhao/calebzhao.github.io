---
title: 深入理解jvm
date: 2019-12-30 08:22:37
tags:
- java
- jvm
categories: java基础
---

# 前言

学习过springboot的都知道，在Springboot的main入口函数中调用SpringApplication.run(DemoApplication.class,args)函数便可以启用SpringBoot应用程序，跟踪一下SpringApplication源码可以发现，最终还是调用了SpringApplication的动态run函数。

下面以SpringBoot2.2.1.RELEASE为例简单分析一下运行过程。



我们在编写一个spring boot应用时通常启动的方式是通过```SpringApplication.run(xxx.class, args)```来启动的，

```java
@SpringBootApplication
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```



# 第一步 分析 SpringApplication构造函数

```SpringApplication```源码：

```java
public class SpringApplication {
    
    public SpringApplication(Class<?>... primarySources) {
            this(null, primarySources);
    }
    
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 1：判断web环境
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
       
        // 2、关键代码：加载classpath下META-INF/spring.factories中key为ApplicationContextInitializer的自动化配置类的名称
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        
       // 3、关键代码：加载classpath下META-INF/spring.factories中key为ApplicationListener的自动化配置类的名称
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        
        //4：推断main方法所在的类
		this.mainApplicationClass = deduceMainApplicationClass();
	}

    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
            return run(new Class<?>[] { primarySource }, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
            return new SpringApplication(primarySources).run(args);
    }
}
```

具体逻辑分析：

## deduceWebApplicationType()

SpringApplication构造函数中首先初始化应用类型，根据加载相关类路径判断应用类型，具体逻辑如下：

WebApplicationType源码如下：

```java
// 应用类型
public enum WebApplicationType {
    // 非servlet应用、reactive应用
	NONE,
    // servlet应用
    SERVLET,
    // reactive反应式应用
    REACTIVE;
    
    private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

	private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

	private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";

	private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

    // 从类路径推倒应用类型
    static WebApplicationType deduceFromClasspath() {
        // 当类路径中存在WEBFLUX_INDICATOR_CLASS
        // 并且不存在WEBMVC_INDICATOR_CLASS时
        // 且不存在JERSEY_INDICATOR_CLASS时
        if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
            && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
            // 是reactive反应式程序
            return WebApplicationType.REACTIVE;
        }
        　// 当加载的类路径中不包含SERVLET_INDICATOR_CLASSES中定义的任何一个类时，返回标准应用
        for (String className : SERVLET_INDICATOR_CLASSES) {
            if (!ClassUtils.isPresent(className, null)) {
                // 是标准应用
                return WebApplicationType.NONE;
            }
        }
        // 加载的类路径中包含了SERVLET_INDICATOR_CLASSES中定义的所有类型则判断为web应用
        return WebApplicationType.SERVLET;
    }
}
```



## setInitializers初始化属性initializers

```setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));```

初始化initializers属性，加载classpath下```META-INF/spring.factories```中配置的ApplicationContextInitializer

此处getSpringFactoriesInstances方法入参type=ApplicationContextInitializer.class



首先看```getSpringFactoriesInstances(ApplicationContextInitializer.class)```这个方法的实现，源码如下：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
}
```

调用了重载的方法，继续看```getSpringFactoriesInstances(type, new Class<?>[] {})```的实现，源码如下：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    
    // SpringFactoriesLoader.loadFactoryNames()方法将会
    // 从calssptah下的META-INF/spring.factories中读取 key为
    // org.springframework.context.ApplicationContextInitializer的值，
    // 并以集合形式返回
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 根据返回names集合逐个实例化,也就是初始化各种ApplicationContextInitializer,
    // 这些Initializer实际是在Spring上下文ApplicationContext执行refresh前调用
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 对上面创建的实例排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

其中上面代码用到的```SpringFactoriesLoader.loadFactoryNames(xx.class)```的具体实现见另一篇文章[springboot2.2自动注入文件spring.factories如何加载详解](https://calebzhao.github.io/2019/12/29/1、springboot2-2自动注入文件spring-factories如何加载详解/)



##  setListeners 初始化属性listeners

setListeners 初始化属性listeners，加载classpath下```META-INF/spring.factories```中配置的```ApplicationListener```，此处入参为```getSpringFactoriesInstances```方法入参type= ApplicationListener.class



```getSpringFactoriesInstances(ApplicationListener.class, new Class<?>[] {})```源码如下

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    
    // SpringFactoriesLoader.loadFactoryNames()方法将会
    // 从calssptah下的META-INF/spring.factories中读取 key为
    // org.springframework.context.ApplicationListener的值，
    // 并以集合形式返回
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 根据返回names集合逐个实例化,也就是初始化各种ApplicationListener,作用是用来监听ApplicationEvent
    // 这些Initializer实际是在Spring上下文ApplicationContext执行refresh前调用
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 对上面创建的实例排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```



## deduceMainApplicationClass()

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```





# 第二步 分析 SpringApplication中 run方法

SpringApplication的run方法代码如下：

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
     // 设置系统变量java.awt.headless
    configureHeadlessProperty();
    
     // 1:加载classpath下面的META-INF/spring.factories SpringApplicationRunListener
    SpringApplicationRunListeners listeners = getRunListeners(args);
    
    // 2：执行所有runlistener的starting方法，实际上发布一个【ApplicationStartingEvent】事件
    listeners.starting();
    try {
        // 3：实例化ApplicationArguments对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
       
        // 4： 创建Environment （web环境 or 标准环境）+配置Environment，主要是把run方法的参数配置
        // 到Environment  发布【ApplicationEnvironmentPreparedEvent】事件
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        // 打印banner，SpringBoot启动时，控制台输出的一个歪歪扭扭的很不清楚的Spring几个大字母，
        // 也可以自定义，参考博客：http://majunwei.com/view/201708171646079868.html
        Banner printedBanner = printBanner(environment);
        
        // 5: 根据不同environment实例化context
        context = createApplicationContext();
        // 异常处理
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                      new Class[] { ConfigurableApplicationContext.class }, context);
      
        // 6: 上下文相关预处理，发布【ApplicationPreparedEvent】事件
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        
        // 7: 执行context的refresh，并且调用context的registerShutdownHook方法
        refreshContext(context);
        
        　// 8:空方法
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 9:执行所有runlisteners的started方法，发布【ApplicationStartedEvent】事件
        listeners.started(context);
        
        // 10: 遍历执行CommandLineRunner和ApplicationRunner
     　 // 如果需要在SpringBoot应用启动后运行一些特殊的逻辑，可以通过实现ApplicationRunner
        // 或CommandLineRunner接口中的run方法，该自定义类的run方法会在此处统一调用
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

