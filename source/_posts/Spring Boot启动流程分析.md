---
title: Spring Boot启动流程分析
date: 2019-12-30 08:22:37
tags: spring boot
categories: spring boot
---

> 有道无术,术可求;有术无道,止于术

# 1、前言

学习过springboot的都知道，在Springboot的main入口函数中调用SpringApplication.run(DemoApplication.class,args)函数便可以启用SpringBoot应用程序，跟踪一下SpringApplication源码可以发现，最终还是调用了SpringApplication的动态run函数。

下面以SpringBoot2.2.1.RELEASE为例简单分析一下运行过程。



我们在编写一个spring boot应用时通常启动的方式是通过```SpringApplication.run(xxx.class, args)```来启动的

```java
@SpringBootApplication
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```



# 2、分析 SpringApplication构造函数

`SpringApplication`源码：

```java
public class SpringApplication {

    public SpringApplication(Class<?>... primarySources) {
        this(null, primarySources);
    }

    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        // 2.1、赋值资源加载器
        this.resourceLoader = resourceLoader;
        // 2.2、断言主要加载资源类不能为 null，否则报错
        Assert.notNull(primarySources, "PrimarySources must not be null");
        // 2.3、初始化主要加载资源类集合并去重
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 2.4、推断当前应用类型是servlet环境、reactive反应式环境、标准环境
        this.webApplicationType = WebApplicationType.deduceFromClasspath();

        // 2.5、关键代码：加载classpath下META-INF/spring.factories中key为ApplicationContextInitializer的自动化配置类的名称
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

        // 2.6、关键代码：加载classpath下META-INF/spring.factories中key为ApplicationListener的自动化配置类的名称
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

        //2.7、推断main方法所在的类
        this.mainApplicationClass = deduceMainApplicationClass();
    }

    // 这个方法就是SpringApplication.run(ClientApplication.class, args);这段代码调用的方法
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class<?>[] { primarySource }, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return new SpringApplication(primarySources).run(args);
    }
}
```

可知这个构造器类的初始化包括以下 7 个过程，下面逐一分析每个步骤的源码:

## 2.1、赋值资源加载器
```java
this.resourceLoader = resourceLoader;
```

## 2.2、断言主要加载资源类不能为 null，否则报错
```java
Assert.notNull(primarySources, "PrimarySources must not be null");
```

## 2.3、初始化主要加载资源类集合并去重
```java
this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
```

## 2.4、推断当前 WEB 应用类型
```java
this.webApplicationType = deduceWebApplicationType();
```
来看下 deduceWebApplicationType 方法和相关的源码：

```java
// 应用类型
public enum WebApplicationType {
    // 非servlet应用、非reactive应用
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



## 2.5、设置ApplicationContextInitializer初始化器

```
setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));
```

初始化initializers属性，加载classpath下```META-INF/spring.factories```中配置的ApplicationContextInitializer。

`ApplicationContextInitializer`的作用是什么？源码如下。

```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    /**
     * Initialize the given application context.
     * @param applicationContext the application to configure
     */
    void initialize(C applicationContext);

}
```

用来初始化指定的 Spring 应用上下文，如注册属性资源、激活 Profiles 等。

来看下 setInitializers 方法源码，其实就是初始化一个 ApplicationContextInitializer 应用上下文初始化器实例的集合。
```java
public void setInitializers(Collection<? extends ApplicationContextInitializer<?>> initializers) {
    this.initializers = new ArrayList<>();
    this.initializers.addAll(initializers);
}
```

再来看下这个初始化 ```getSpringFactoriesInstances``` 方法和相关的源码：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

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
这个方法会尝试从类路径的META-INF/spring.factories处读取相应配置文件，然后进行遍历，读取配置文件中Key   为：```org.springframework.context.ApplicationContextInitializer```的value。以spring-boot-autoconfigure这个包为例，它的```META-INF/spring.factories```部分定义如下所示：
```properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```

其中上面代码用到的```SpringFactoriesLoader.loadFactoryNames(xx.class)```的具体实现见另一篇文章[springboot2.2自动注入文件spring.factories如何加载详解](https://calebzhao.github.io/2019/12/29/1、springboot2-2自动注入文件spring-factories如何加载详解/)


## 2.6、设置ApplicationListener监听器
```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

setListeners 初始化属性listeners，加载classpath下```META-INF/spring.factories```中配置的```ApplicationListener```，此处入参为```getSpringFactoriesInstances```方法入参type= ApplicationListener.class

ApplicationListener 的作用是什么？源码如下。
```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    /**
     * Handle an application event.
     * @param event the event to respond to
     */
    void onApplicationEvent(E event);

}
```

看源码，这个接口继承了 JDK 的``` java.util.EventListener``` 接口，实现了观察者模式，它一般用来定义感兴趣的事件类型，事件类型限定于 ApplicationEvent 的子类，这同样继承了 JDK 的 ```java.util.EventObject``` 接口。

设置监听器和设置初始化器调用的方法是一样的，只是传入的类型不一样，设置监听器的接口类型为：```getSpringFactoriesInstances```，对应的```spring-boot-autoconfigure-2.2.1.RELEASE.jar!/META-INF/spring.factories``` 文件配置内容请见下方。

```properties
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```

可以看出目前只有一个 `BackgroundPreinitializer` 监听器。

## 2.7、推断主入口应用类
```java
deduceMainApplicationClass()
```

deduceMainApplicationClass()方法的源码实现如下：
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
这个推断入口应用类的方式有点特别，通过构造一个运行时异常，再遍历异常栈中的方法名，获取方法名为 main 的栈帧，从来得到入口类的名字再返回该类。




# 3、 分析 SpringApplication中 run方法

SpringApplication的run方法代码如下：

```java
public ConfigurableApplicationContext run(String... args) {
    // 1、创建并启动计时监控类
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // 2、初始化应用上下文和异常报告集合
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

    // 3、设置系统属性 `java.awt.headless` 的值，默认值为：true
    configureHeadlessProperty();

    // 4、加载classpath下面的META-INF/spring.factories中key为SpringApplicationRunListener的配置
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 执行所有runlistener的starting方法，实际上发布一个【ApplicationStartingEvent】事件
    // 内部会发布ApplicationStartingEvent事件，有以下监听器监听了该事件（按执行顺序列出）：
    // 括号中的代码表示该listener时是在哪个项目的spring.factories中声明的 
    // org.springframework.boot.context.logging.LoggingApplicationListener  （spring-boot-2.2.1.RELEASE.jar）
    // org.springframework.boot.autoconfigure.BackgroundPreinitializer       （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
    // org.springframework.boot.context.config.DelegatingApplicationListener   （spring-boot-2.2.1.RELEASE.jar）
    // org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener   （spring-boot-2.2.1.RELEASE.jar）
    listeners.starting();

    try {
        // 5、实例化ApplicationArguments对象，初始化默认应用参数类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // 6、 创建Environment （web环境 or 标准环境）+配置Environment，主要是把run方法的参数配置
        // 到Environment  发布【ApplicationEnvironmentPreparedEvent】事件，依次执行如下的listener：
        // 括号中的代码表示该listener时是在哪个项目的spring.factories中声明的 
        // org.springframework.cloud.bootstrap.BootstrapApplicationListener  (spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.cloud.bootstrap.LoggingSystemShutdownListener  (spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.config.ConfigFileApplicationListener (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.config.AnsiOutputApplicationListener  (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.logging.LoggingApplicationListener   (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.autoconfigure.BackgroundPreinitializer       (spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.logging.ClasspathLoggingApplicationListener (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.config.DelegatingApplicationListener    (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
        // org.springframework.boot.context.FileEncodingApplicationListener         (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)

        // 这段代码非常非常关键，spring cloud与spring boot的整合就是在这里开始的
        // spring cloud的BootstrapApplicationListener实现了ApplicationListener接口，并且接收ApplicationEnvironmentPreparedEvent事件
        // BootstrapApplicationListener中会创建一个新的SpringApplication，又会调用新创建的SpringApplication.run()方法产生id为bootstrap的
        // 父ApplicationContext

        // 返回类型可能是StandardServletEnvironment(spring boot servlet环境)或StandardEnvironment(spring cloud使用)，
        // 下面列出environment中属性源的名称，属性源按照在ConfigurableEnvironment中的顺序包括:
        // 【springApplicationCommandLineArgs】 // 有命令参数才有该属性源
        //  configurationProperties 
        // 【bootstrap】 //spring cloud中的， spring boot不存在该属性源
        // 【servletConfigInitParams】// servlet应用存在该属性源， spring cloud不存在
        // 【servletContextInitParams】// servlet应用存在该属性源， spring cloud不存在
        //  systemProperties jvm系统属性源
        //  systemEnvironment 系统环境变量属性源
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);

        // 7、打印banner，SpringBoot启动时，控制台输出的一个歪歪扭扭的很不清楚的Spring几个大字母，
        // 也可以自定义，参考博客：http://majunwei.com/view/201708171646079868.html
        Banner printedBanner = printBanner(environment);

        // 8、 根据不同environment创建应用上下文（返回ConfigurableApplicationContext的子类实例）
        context = createApplicationContext();

        // 9、通过SpringFactoriesLoader检索META-INF/spring.factories中的SpringBootExceptionReporter，
        // 获取并实例化异常分析器
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                         new Class[] { ConfigurableApplicationContext.class }, context);

        // 10、 上下文相关预处理，发布【ApplicationPreparedEvent】事件 
        // 为ApplicationContext加载environment，之后逐个执行ApplicationContextInitializer的initialize()方法来进一步封装ApplicationContext，
        // 并调用所有的SpringApplicationRunListener的contextPrepared()方法，【EventPublishingRunListener只提供了一个空的contextPrepared()方法】，
        // 之后初始化IoC容器，并调用SpringApplicationRunListener的contextLoaded()方法，广播ApplicationContext的IoC加载完成，
        // 这里就包括通过@EnableAutoConfiguration导入的各种自动配置类。
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);

        // 11、 执行context的refresh，并且调用context的registerShutdownHook方法
        refreshContext(context);

        // 12、应用上下文刷新后置处理，是一个空方法
        afterRefresh(context, applicationArguments);

        // 13、停止计时监控类
        stopWatch.stop();

        // 14、输出日志记录执行主类名、时间信息
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }

        // 15、执行所有runlisteners的started方法，发布【ApplicationStartedEvent】事件(应用已经启动的事件)
        listeners.started(context);

        // 16、遍历所有注册的ApplicationRunner和CommandLineRunner，并执行其run()方法。
        // 我们可以实现自己的ApplicationRunner或者CommandLineRunner，来对SpringBoot的启动过程进行扩展。
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 17、遍历前面14行获得SpringApplicationRunListeners局部变量中所维护的listeners属性,
        // 执行所有SpringApplicationRunListener的running方法，发布【ApplicationReadyEvent】事件（应用已经启动完成的监听事件）
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }

    // 18、返回应用上下文
    return context;
}
```

其实这个方法我们可以简单的总结下步骤为 ：

1. 配置属性 
2. 获取监听器，发布应用开始启动事件 
3. 初始化输入参数 
4. 配置环境，输出banner 
5. 创建上下文 
6. 预处理上下文 
7. 刷新上下文
8. 刷新上下文的后处理，空实现
9. 发布应用已经启动事件
10. 发布应用启动完成事件



所以，我们可以按以下几步来分解 run 方法的启动过程。

## 3.1、创建并启动StopWatch

创建并启动计时监控类

```java
StopWatch stopWatch = new StopWatch();
stopWatch.start();
```

来看下这个计时监控类 StopWatch 的相关源码：

```java
public class StopWatch {

    public void start() throws IllegalStateException {
        start("");
    }

    public void start(String taskName) throws IllegalStateException {
        if (this.currentTaskName != null) {
            throw new IllegalStateException("Can't start StopWatch: it's already running");
        }
        this.currentTaskName = taskName;
        this.startTimeNanos = System.nanoTime();
    }

}
```

首先记录了当前任务的名称，默认为空字符串，然后记录当前 Spring Boot 应用启动的开始时间（单位纳秒）。

## 3.2、初始化SpringBootExceptionReporter
初始化应用上下文和异常报告集合

```java
ConfigurableApplicationContext context = null;
Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
```

## 3.3、设置系统属性 `java.awt.headless` 的值

```java
configureHeadlessProperty();
```

设置该默认值为：true，Java.awt.headless = true 有什么作用？

> 对于一个 Java 服务器来说经常要处理一些图形元素，例如地图的创建或者图形和图表等。这些API基本上总是需要运行一个X-server以便能使用AWT（Abstract Window Toolkit，抽象窗口工具集）。然而运行一个不必要的 X-server 并不是一种好的管理方式。有时你甚至不能运行 X-server,因此最好的方案是运行 headless 服务器，来进行简单的图像处理。
>
> 参考：www.cnblogs.com/princessd8251/p/4000016.html

## 3.4、创建SpringApplicationRunListeners并发布ApplicationStartingEvent

### 3.4.1、getRunListeners()方法
```java
SpringApplicationRunListeners listeners = getRunListeners(args);
// 内部会发布ApplicationStartingEvent事件，有以下监听器监听了该事件（按执行顺序列出）：
// org.springframework.boot.context.logging.LoggingApplicationListener
// org.springframework.boot.autoconfigure.BackgroundPreinitializer
// org.springframework.boot.context.config.DelegatingApplicationListener
// org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
listeners.starting();
```

来看下创建 SpringApplicationRunListeners运行监听器相关的源码：

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    // SpringApplicationRunListener的实现实际只有EventPublishingRunListener
    return new SpringApplicationRunListeners(logger,
                                             getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

可以看到返回了SpringApplicationRunListeners对象，

### 3.4.2、创建SpringApplicationRunListeners

返回的SpringApplicationRunListeners这个类的源码如下：

```java
class SpringApplicationRunListeners {
    private final Log log;

    private final List<SpringApplicationRunListener> listeners;

    /**
    *
    * @param listeners 这个集合实际只有EventPublishingRunListener一个类的实例
    */
    SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
        this.log = log;
        this.listeners = new ArrayList<>(listeners);
    }

    void starting() {
        for (SpringApplicationRunListener listener : this.listeners) {
            listener.starting();
        }
    }
}
```

创建逻辑和之前实例化初始化器和监听器的一样，一样调用的是 `getSpringFactoriesInstances` 方法来获取配置的监听器名称并实例化所有的类。

SpringApplicationRunListener 所有监听器配置在 `spring-boot-2.2.1.RELEASE.jar!/META-INF/spring.factories` 这个配置文件里面。

  ```properties
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
  ```

可以看到SpringApplicationRunListener的值是```org.springframework.boot.context.event.EventPublishingRunListener```这个类，所以创建SpringApplicationRunListeners这个对象时构造方法传递的2个参数就是这个EventPublishingRunListener的实例。

### 3.4.3、listeners.starting()

所以```listeners.starting();```这行代码的实现SpringApplicationRunListeners.starting()方法中遍历的属性```this.listeners```实际就只有1个```EventPublishingRunListener```

```java
class SpringApplicationRunListeners{
    private final List<SpringApplicationRunListener> listeners;
    
    void starting() {
        for (SpringApplicationRunListener listener : this.listeners) {
            listener.starting();
        }
    }
}
```


接下来看```EventPublishingRunListener```类的starting方法的源码实现。

### 3.4.4、EventPublishingRunListener.starting()

```java
/**
* 用于发布多种SpringApplicationEvent事件。
* 在上下文ApplicationContext实际refresh之前使用ApplicationEventMulticaster来触发事件。
* 
* 
*/
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

    // SpringApplication.run(xxx.class, args)运行时内部创建的SpringApplication实例，
    // 构造方法实例化时设置了List<ApplicationContextInitializer<?>> initializers及List<ApplicationListener<?>> listeners属性
    private final SpringApplication application;

    // SpringApplication.run(xxx.class, args)传入的args参数
    private final String[] args;

    // 在构造方法中初始化的，实例类型为SimpleApplicationEventMulticaster
    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        //创建了简单应用程序事件发布多播器SimpleApplicationEventMulticaster
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        // 观察者模式的典型应用。
        // 将所有的ApplicationListener观察者加入到initialMulticaster中，当被观察者(SpringApplication)向观察者(ApplicationListener)
        // 发布事件时，内部遍历所有的ApplicationListener(观察者)找出对发布的事件(ApplicationEvent)感兴趣的ApplicationListener(观察者)
        // 然后调用ApplicationListener(观察者)的onEvent(ApplicationEvent event)方法回调观察者
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.initialMulticaster.addApplicationListener(listener);
        }
    }

    @Override
    public void starting() {
        // 向所有注册的ApplicationListener(观察者)发布ApplicationStartingEvent事件，
        // 注意initialMulticaster内部会找出对发布的事件(ApplicationEvent)感兴趣的ApplicationListener(观察者)
        // 并回调观察者的onEvent(ApplicationEvent event)方法
        this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
    }

    ...省略发布其他事件的方法
}
```

需要注意的是这里是一个典型的**观察者模式**的应用，```SpringApplication```是被观察者，```ApplicationListener```接口的实现是观察者，spring启动时从```spring.factories```文件中根据找出所有的key为```org.springframework.context.ApplicationListener```的```ApplicationListener```接口的实现(观察者)，然后在```EventPublishingRunListener```类的构造方法将所有的ApplicationListener接口的实现(观察者)注册到类型为```SimpleApplicationEventMulticaster```的initialMulticaster属性中，而```SpringApplication```的run方法中又持有了EventPublishingRunListener这个类的引用(```  SpringApplicationRunListeners listeners = getRunListeners(args);```)，所以在```SpringApplication```是中可以通过listeners.starting()方法调用委托给真正的```SimpleApplicationEventMulticaster```进行事件发布。

### 3.4.5、SimpleApplicationEventMulticaster.multicastEvent(ApplicationEvent event)

SimpleApplicationEventMulticaster类的作用时真正的发布事件(ApplicationEvent event)。

SimpleApplicationEventMulticaster类的源码如下：

```java
/**
* ApplicationEventMulticaster接口的简单实现。
* 将所有事件(ApplicationEvent)广播给所有已注册的监听器(ApplicationEvent)，然后由监听器忽略他们不感兴趣的事件，
* 监听器通常对传入的事件执行相应的instanceof检查，来确定该事件是否是自己感兴趣的，如果对该事件不敢兴趣就忽略它，否则进行业务逻辑处理。
*
* 默认情况下所有监听器在调用线程中被执行，但这带来了恶意的侦听器可能会阻塞整个应用的风险，但只增加了最小的开销，可以指定一个可选的
* 任务执行器(Executor taskExecutor)来让监听器(ApplicationListener)在不同的线程中执行，例如这个taskExecutor可以是一个线程池，
* 这样无论ApplicationListener的执行时间多长(比如ApplicationListener中存在耗时的网络请求、性能不好的数据库访问或是否是阻塞操作(比如io、等待CPU)），
* 都不会导致整个应用被阻塞
*/
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
    // 指定ApplicationListener在哪个线程执行器中执行（可以不指定），具体含义参见上面类级别的文档
    @Nullable
    private Executor taskExecutor;

    // 错误处理器，作用参见invokeListener(ApplicationListener<?> listener, ApplicationEvent event)方法内注释
    @Nullable
    private ErrorHandler errorHandler;

    public SimpleApplicationEventMulticaster() {

    }

    // 可以指定ApplicationListener在哪个线程执行器中执行
    public void setTaskExecutor(@Nullable Executor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    // 可以指定ApplicationListener的错误处理器
    public void setErrorHandler(@Nullable ErrorHandler errorHandler) {
        this.errorHandler = errorHandler;
    }



    @Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        // 获得ApplicationListener的执行器（该执行器是可选的）
        Executor executor = getTaskExecutor();
        // 找到对该ApplicationEvent感兴趣的监听器(ApplicationListener)
        for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            // 判断是否指定了ApplicationListener在哪个线程执行器中执行
            if (executor != null) {
                // 在指定的taskExecutor中执行，不会阻塞调用线程
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                // 没有指定taskExecutor，执行在调用线程中执行（可能会阻塞调用线程）
                invokeListener(listener, event);
            }
        }
    }

    protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        // 是否有错误处理器，如果有错误处理器，那么ApplicationListener方法内执行抛出异常时会捕获抛出的异常交由指定的错误处理器去处理
        // 否则spring不处理
        ErrorHandler errorHandler = getErrorHandler();
        if (errorHandler != null) {
            try {
                doInvokeListener(listener, event);
            }
            catch (Throwable err) {
                // 交由错误处理器处理异常
                errorHandler.handleError(err);
            }
        }
        else {
            // 如果ApplicationListener方法执行时产生异常，则不处理，由上级处理
            doInvokeListener(listener, event);
        }
    }

    private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
        try {
            // onApplicationEvent方法就是我们平时使用ApplicationLisenter接口时要实现的方法，回调观察者
            listener.onApplicationEvent(event);
        }
        catch (ClassCastException ex) {
            String msg = ex.getMessage();
            if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
                // Possibly a lambda-defined listener which we could not resolve the generic event type for
                // -> let's suppress the exception and just log a debug message.
                Log logger = LogFactory.getLog(getClass());
                if (logger.isTraceEnabled()) {
                    logger.trace("Non-matching event type for listener: " + listener, ex);
                }
            }
            else {
                throw ex;
            }
        }
    }
}
```



## 3.5、初始化默认应用参数类

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
```

DefaultApplicationArguments类的源码如下：

```java
public class DefaultApplicationArguments implements ApplicationArguments {

    private final Source source;

    private final String[] args;

    public DefaultApplicationArguments(String... args) {
        Assert.notNull(args, "Args must not be null");
        this.source = new Source(args);
        this.args = args;
    }

    private static class Source extends SimpleCommandLinePropertySource {

        Source(String[] args) {
            super(args);
        }

        @Override
        public List<String> getNonOptionArgs() {
            return super.getNonOptionArgs();
        }

        @Override
        public List<String> getOptionValues(String name) {
            return super.getOptionValues(name);
        }

    }
}
```

从构造方法可以看到最主要的操作就是```this.source = new Source(args);```这一行代码，而Souce类是一个内部内，Source的构造方法中直接调用了父类SimpleCommandLinePropertySource的构造方法，下面看```SimpleCommandLinePropertySource```类构造方法的实现细节：

```java
public class SimpleCommandLinePropertySource extends CommandLinePropertySource<CommandLineArgs> {
    
	public SimpleCommandLinePropertySource(String... args) {
		super(new SimpleCommandLineArgsParser().parse(args));
	}
}
```

可以看到创建了SimpleCommandLineArgsParser类的实例，调用了它的parse(String... args)方法，从类名和方法命名可以就应该能猜到这个类是用来解析命令行参数的。

SimpleCommandLineArgsParser类源码如下：

```java
class SimpleCommandLineArgsParser {

    public CommandLineArgs parse(String... args) {
        CommandLineArgs commandLineArgs = new CommandLineArgs();
        // 比如命令 java --server.port=8888 --spring.profiles.active=prod -jar app.jar
        for (String arg : args) {
            // 参数是否以--打头，比如参数 --server.port=8888
            if (arg.startsWith("--")) {
                // 截取--字符串后面的字符串，比如--server.port=8888变成server.port=8888
                String optionText = arg.substring(2, arg.length());
                String optionName;
                String optionValue = null;
                // 是否有值
                if (optionText.contains("=")) {
                    // 等号前面的字符串作为name
                    optionName = optionText.substring(0, optionText.indexOf('='));
                    // 等号后面的字符串作为value
                    optionValue = optionText.substring(optionText.indexOf('=')+1, optionText.length());
                }
                else {
                    // --后面的整个字符串作为name
                    optionName = optionText;
                }
                if (optionName.isEmpty() || (optionValue != null && optionValue.isEmpty())) {
                    throw new IllegalArgumentException("Invalid argument syntax: " + arg);
                }

                // 存储name和value到CommandLineArgs中
                commandLineArgs.addOptionArg(optionName, optionValue);
            }
            else {
                commandLineArgs.addNonOptionArg(arg);
            }
        }
        return commandLineArgs;
    }
}
```

到这里```ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);```这一行代码的作用就分析完了，可以看到做的事情就是解析命令行参数。



## 3.6、创建Environment并发布ApplicationEnvironmentPreparedEvent

```
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
configureIgnoreBeanInfo(environment);
```

下面我们主要来看下准备环境的 `prepareEnvironment` 源码：

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                   ApplicationArguments applicationArguments) {
    // 3.6.1、 获取（或者创建）应用ConfigurableEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();

    // 3.6.2、 配置应用环境（启动参数绑定到ConfigurableEnvironment中、绑定ConfigurableConversionService到ConfigurableEnvironment中）
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 3.6.3、
    ConfigurationPropertySources.attach(environment);
    
    // 3.6.4、这里的listeners是SpringApplicationRunListeners，发布ApplicationEnvironmentPreparedEvent事件
    
    // 这段代码非常非常重要，spring cloud与spring boot的整合就是在这里开始的
    // spring cloud的BootstrapApplicationListener实现了ApplicationListener接口，并且接收ApplicationEnvironmentPreparedEvent事件
    // BootstrapApplicationListener中会创建一个新的SpringApplication，又会调用新创建的SpringApplication.run()方法产生id为bootstrap的
    // 父ApplicationContext
    listeners.environmentPrepared(environment);
    
    // 3.6.5、绑定ConfigurableEnvironment到当前的SpringApplication实例中
    bindToSpringApplication(environment);
    
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                                                                                               deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

**上面的代码需要重点关注，和spring cloud的整合在这里开始**。

### 3.6.1、获取（或者创建）ConfigurableEnvironment

```java
 ConfigurableEnvironment environment = getOrCreateEnvironment();
```

getOrCreateEnvironment()方法实现如下：
```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
    }
}
```

这里分为标准 Servlet 环境和标准环境。

### 3.6.2、 配置ConfigurableEnvironment
```java
 configureEnvironment(environment, applicationArguments.getSourceArgs());
```

configureEnvironment()实现如下：
```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    // 3.6.2.1、创建ConversionService并存到Environment中
    // SpringApplication的成员变量，默认值为true。
    if (this.addConversionService) {
        // 创建ApplicationConversionService(类型转换器)并注册默认的Converter、Formatter类型转换器实现，spring mvc中经常使用ConversionService
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        // 将ConversionService存到ConfigurableEnvironment中
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    // 3.6.2.2、将命令行的属性添加到Environment中
    configurePropertySources(environment, args);
    // 3.6.2.3、将additionalProfiles合并到Environment中，设置激活的profile
    configureProfiles(environment, args);
}
```

#### 3.6.2.1、创建ConversionService并存到Environment中

- ApplicationConversionService.getSharedInstance()源码如下：

```java
/**
* 类型转换器服务，用于将
*/
public class ApplicationConversionService extends FormattingConversionService {	
    private static volatile ApplicationConversionService sharedInstance;

    public ApplicationConversionService() {
        this(null);
    }

    public ApplicationConversionService(StringValueResolver embeddedValueResolver) {
        if (embeddedValueResolver != null) {
            setEmbeddedValueResolver(embeddedValueResolver);
        }
        configure(this);
    }

    public static ConversionService getSharedInstance() {
        ApplicationConversionService sharedInstance = ApplicationConversionService.sharedInstance;
        if (sharedInstance == null) {
            synchronized (ApplicationConversionService.class) {
                sharedInstance = ApplicationConversionService.sharedInstance;
                if (sharedInstance == null) {
                    sharedInstance = new ApplicationConversionService();
                    ApplicationConversionService.sharedInstance = sharedInstance;
                }
            }
        }
        return sharedInstance;
    }

    public static void configure(FormatterRegistry registry) {
        DefaultConversionService.addDefaultConverters(registry);
        DefaultFormattingConversionService.addDefaultFormatters(registry);
        addApplicationFormatters(registry);
        addApplicationConverters(registry);
    }

    public static void addApplicationFormatters(FormatterRegistry registry) {
        registry.addFormatter(new CharArrayFormatter());
        registry.addFormatter(new InetAddressFormatter());
        registry.addFormatter(new IsoOffsetFormatter());
    }

    public static void addApplicationConverters(ConverterRegistry registry) {
        addDelimitedStringConverters(registry);
        registry.addConverter(new StringToDurationConverter());
        registry.addConverter(new DurationToStringConverter());
        registry.addConverter(new NumberToDurationConverter());
        registry.addConverter(new DurationToNumberConverter());
        registry.addConverter(new StringToDataSizeConverter());
        registry.addConverter(new NumberToDataSizeConverter());
        registry.addConverter(new StringToFileConverter());
        registry.addConverterFactory(new LenientStringToEnumConverterFactory());
        registry.addConverterFactory(new LenientBooleanToEnumConverterFactory());
    }
}
```

可以看到getSharedInstance()方法就是一个典型的双重判断单例模式的实现，只是需要注意这里sharedInstance变量是成员变量使用volatile进行了修饰，禁止指令重排序及保证内存可见性，ApplicationConversionService的构造方法中最终调用了```configure(FormatterRegistry registry)```方法

从方法实现看默认注册了一些Convert、Formater接口的实现。

- DefaultConversionService.addDefaultConverters(registry);源码如下：

```java
public class DefaultConversionService extends GenericConversionService {
    public DefaultConversionService() {
        addDefaultConverters(this);
    }

    public static void addDefaultConverters(ConverterRegistry converterRegistry) {
        addScalarConverters(converterRegistry);
        addCollectionConverters(converterRegistry);

        converterRegistry.addConverter(new ByteBufferConverter((ConversionService) converterRegistry));
        converterRegistry.addConverter(new StringToTimeZoneConverter());
        converterRegistry.addConverter(new ZoneIdToTimeZoneConverter());
        converterRegistry.addConverter(new ZonedDateTimeToCalendarConverter());

        converterRegistry.addConverter(new ObjectToObjectConverter());
        converterRegistry.addConverter(new IdToEntityConverter((ConversionService) converterRegistry));
        converterRegistry.addConverter(new FallbackObjectToStringConverter());
        converterRegistry.addConverter(new ObjectToOptionalConverter((ConversionService) converterRegistry));
    }

    ...省略部分代码
}
```



- DefaultFormattingConversionService.addDefaultFormatters(registry);实现源码如下：

```java
public class DefaultFormattingConversionService extends FormattingConversionService {

    private static final boolean jsr354Present;

    private static final boolean jodaTimePresent;

    static {
        ClassLoader classLoader = DefaultFormattingConversionService.class.getClassLoader();
        jsr354Present = ClassUtils.isPresent("javax.money.MonetaryAmount", classLoader);
        jodaTimePresent = ClassUtils.isPresent("org.joda.time.LocalDate", classLoader);
    }

    public static void addDefaultFormatters(FormatterRegistry formatterRegistry) {
        // Default handling of number values
        formatterRegistry.addFormatterForFieldAnnotation(new NumberFormatAnnotationFormatterFactory());

        // Default handling of monetary values
        if (jsr354Present) {
            formatterRegistry.addFormatter(new CurrencyUnitFormatter());
            formatterRegistry.addFormatter(new MonetaryAmountFormatter());
            formatterRegistry.addFormatterForFieldAnnotation(new Jsr354NumberFormatAnnotationFormatterFactory());
        }

        // Default handling of date-time values

        // just handling JSR-310 specific date and time types
        new DateTimeFormatterRegistrar().registerFormatters(formatterRegistry);

        if (jodaTimePresent) {
            // handles Joda-specific types as well as Date, Calendar, Long
            new JodaTimeFormatterRegistrar().registerFormatters(formatterRegistry);
        }
        else {
            // regular DateFormat-based Date, Calendar, Long converters
            new DateFormatterRegistrar().registerFormatters(formatterRegistry);
        }
    }

    ...省略部分代码
}
```



#### 3.6.2.2、将命令行的属性添加到Environment中

`configurePropertySources`方法源码如下：


```java
public class SpringApplication {


    // ...省略代码

    // 是否添加命令行参数到ConfigurableEnvironment中
    private boolean addCommandLineProperties = true;

    // 默认属性，如果有则会合并到ConfigurableEnvironment中
    private Map<String, Object> defaultProperties;

    public void setDefaultProperties(Map<String, Object> defaultProperties) {
        this.defaultProperties = defaultProperties;
    }


    protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {

        if (this.addConversionService) {
            ConversionService conversionService = ApplicationConversionService.getSharedInstance();
            environment.setConversionService((ConfigurableConversionService) conversionService);
        }
        // 将命令行的属性添加到Environment中
        configurePropertySources(environment, args);
        configureProfiles(environment, args);
    }

    // ...省略部分代码

    /**
    * 创建一个基于命令行属性的PropertySource并将其添加到ConfigurableEnvironment中
    *
    * @param environment 环境对象，根据应用类型可能是StandardEnvironment、StandardServletEnvironment、StandardReactiveWebEnvironment
    * @param args 要被添加的命令行参数数据，如 ['--server.port=9090', '--spring.profiles.active=prod']
    */
    protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
        // 获得AbstractEnvironment中的propertySources属性，代表可变属性源，
        // MutablePropertySources类中包含addFirst、addBefore、addLast、remove、replace等操纵属性源的方法
        MutablePropertySources sources = environment.getPropertySources();
        // 判断是否有默认属性
        if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
            // 如果有默认属性，则将其添加到environment中
            sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
        }
        // 判断是否添加命令行参数到ConfigurableEnvironment中，默认true
        if (this.addCommandLineProperties && args.length > 0) {
            // 命令行属性源名称
            String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
            // 判断ConfigurableEnvironment中是否已经存在该属性源
            if (sources.contains(name)) {
                // 根据name获取ConfigurableEnvironment中已存在的属性源
                PropertySource<?> source = sources.get(name);

                // 声明1个有属性源复合功能的属性源，当复合属性源中维护的多个属性源都有相同的key时，位置靠前的属性源中的值优先返回
                CompositePropertySource composite = new CompositePropertySource(name);
                // 将参数args属性作为第1个属性源，由于它的位置在第1个，所以它的优先级最高
                composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
                // 将ConfigurableEnvironment中已存在的属性源作为第2个属性源，优先级比上面一行的SimpleCommandLinePropertySource低
                composite.addPropertySource(source);

                //  将ConfigurableEnvironment中已存在的单一属性源替换为上面创建的复合属性源
                sources.replace(name, composite);
            }
            // ConfigurableEnvironment中不存在该属性源
            else {
                // 将命令行参数添加ConfigurableEnvironment中的(底层维护着MutablePropertySources类型的propertySources属性)
                sources.addFirst(new SimpleCommandLinePropertySource(args));
            }
        }
    }

    ...省略代码
}
```

接下来继续分析```configurePropertySources(environment, args);```这一行后面的代码```configureProfiles(environment, args);```

#### 3.6.2.3、configureProfiles(environment, args);

配置additionalProfiles到Environment中，设置激活的profile

```java
public class SpringApplication {
    // 额外的profile的名称，可以是多个（表示同时启用多个profile）
    private Set<String> additionalProfiles = new HashSet<>();

    ...省略代码

        public void setAdditionalProfiles(String... profiles) {
        this.additionalProfiles = new LinkedHashSet<>(Arrays.asList(profiles));
    }    

    /**
    * 这个方法的作用是将通过代码设置的profile的名称追加到environment中，例如如下实例：
    * SpringApplication app = new SpringApplication(TestApplication.class);
    * app.setAdditionalProfiles("dev", "test");
    * app.run(args);
    */
    protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
        Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
        // environment.getActiveProfiles()获得当前激活的profile的名称集合(返回的是Set<String>类型)
        // 然后加入profiles集合中，此时profiles集合中包含2种profile来源：this.additionalProfiles及environment中的
        profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
        // 设置使用的profile
        environment.setActiveProfiles(StringUtils.toStringArray(profiles));
    }

    ...省略代码
}
```

prepareEnvironment()方法的源码分析完了，现在让我们回到```public ConfigurableApplicationContext run(String... args)```方法中，继续看prepareEnvironment()方法之后的方法```configureIgnoreBeanInfo(environment)```。



### 3.6.3、启动参数绑定到ConfigurableEnvironment中

```java
ConfigurationPropertySources.attach(environment);
```

`ConfigurationPropertySources`类的静态方法`attach(Environment environment)`源码如下：


```java
public final class ConfigurationPropertySources {

    /**
	 * The name of the {@link PropertySource} {@link #adapt adapter}.
	 */
    private static final String ATTACHED_PROPERTY_SOURCE_NAME = "configurationProperties";

    private ConfigurationPropertySources() {
    }

    public static void attach(Environment environment) {
        Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
        MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
        PropertySource<?> attached = sources.get(ATTACHED_PROPERTY_SOURCE_NAME);
        if (attached != null && attached.getSource() != sources) {
            sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
            attached = null;
        }
        if (attached == null) {
            sources.addFirst(new ConfigurationPropertySourcesPropertySource(ATTACHED_PROPERTY_SOURCE_NAME,
                                                                            new SpringConfigurationPropertySources(sources)));
        }
    }
}
```



### 3.6.4、发布ApplicationEnvironmentPreparedEvent事件

```java
// 内部会发布ApplicationEnvironmentPreparedEvent事件，依次执行如下的listener：
// 括号中的代码表示该listener时是在哪个项目的spring.factories中声明的 
// org.springframework.cloud.bootstrap.BootstrapApplicationListener  (spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.cloud.bootstrap.LoggingSystemShutdownListener  (spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.config.ConfigFileApplicationListener (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.config.AnsiOutputApplicationListener  (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.logging.LoggingApplicationListener   (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.autoconfigure.BackgroundPreinitializer       (spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.logging.ClasspathLoggingApplicationListener (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.config.DelegatingApplicationListener    (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
// org.springframework.boot.context.FileEncodingApplicationListener         (spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories)
listeners.environmentPrepared(environment);

```

`SpringApplicationRunListeners`类的`environmentPrepared`方法源码如下：


```java
class SpringApplicationRunListeners {

    private final Log log;

    // SpringApplicationRunListener`的实现类只有EventPublishingRunListener
    private final List<SpringApplicationRunListener> listeners;
    
    // 该构造方法是在SpringApplication#getRunListeners(String[] args)中调用的
    // 传入的listeners集合就只有1个元素EventPublishingRunListener
    SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
		this.log = log;
		this.listeners = new ArrayList<>(listeners);
	}

    void environmentPrepared(ConfigurableEnvironment environment) {
        // this.listeners集合就只有EventPublishingRunListener
        for (SpringApplicationRunListener listener : this.listeners) {
            // 调用EventPublishingRunListener.environmentPrepared(ConfigurableEnvironment environment)方法
            listener.environmentPrepared(environment);
        }
    }
}
```

`SpringApplicationRunListener`的实现类只有`EventPublishingRunListener`，EventPublishingRunListener类的`environmentPrepared`方法的源码如下：



```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

    private final SpringApplication application;

    private final String[] args;

    private final SimpleApplicationEventMulticaster initialMulticaster;

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        
        this.initialMulticaster
            .multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
    }

}
```



#### 3.6.4.1、SimpleApplicationEventMulticaster.multicastEvent(ApplicationEvent event)

SimpleApplicationEventMulticaster类的作用时真正的发布事件(ApplicationEvent event)。

SimpleApplicationEventMulticaster类的源码如下：

```java
/**
* ApplicationEventMulticaster接口的简单实现。
* 将所有事件(ApplicationEvent)广播给所有已注册的监听器(ApplicationEvent)，然后由监听器忽略他们不感兴趣的事件，
* 监听器通常对传入的事件执行相应的instanceof检查，来确定该事件是否是自己感兴趣的，如果对该事件不敢兴趣就忽略它，否则进行业务逻辑处理。
*
* 默认情况下所有监听器在调用线程中被执行，但这带来了恶意的侦听器可能会阻塞整个应用的风险，但只增加了最小的开销，可以指定一个可选的
* 任务执行器(Executor taskExecutor)来让监听器(ApplicationListener)在不同的线程中执行，例如这个taskExecutor可以是一个线程池，
* 这样无论ApplicationListener的执行时间多长(比如ApplicationListener中存在耗时的网络请求、性能不好的数据库访问或是否是阻塞操作(比如io、等待CPU)），
* 都不会导致整个应用被阻塞
*/
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
    // 指定ApplicationListener在哪个线程执行器中执行（可以不指定），具体含义参见上面类级别的文档
    @Nullable
    private Executor taskExecutor;

    // 错误处理器，作用参见invokeListener(ApplicationListener<?> listener, ApplicationEvent event)方法内注释
    @Nullable
    private ErrorHandler errorHandler;

    public SimpleApplicationEventMulticaster() {

    }

    // 可以指定ApplicationListener在哪个线程执行器中执行
    public void setTaskExecutor(@Nullable Executor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    // 可以指定ApplicationListener的错误处理器
    public void setErrorHandler(@Nullable ErrorHandler errorHandler) {
        this.errorHandler = errorHandler;
    }



    @Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        // 获得ApplicationListener的执行器（该执行器是可选的）
        Executor executor = getTaskExecutor();
        // 找到对该ApplicationEvent感兴趣的监听器(ApplicationListener)
        for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            // 判断是否指定了ApplicationListener在哪个线程执行器中执行
            if (executor != null) {
                // 在指定的taskExecutor中执行，不会阻塞调用线程
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                // 没有指定taskExecutor，执行在调用线程中执行（可能会阻塞调用线程）
                invokeListener(listener, event);
            }
        }
    }

    protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        // 是否有错误处理器，如果有错误处理器，那么ApplicationListener方法内执行抛出异常时会捕获抛出的异常交由指定的错误处理器去处理
        // 否则spring不处理
        ErrorHandler errorHandler = getErrorHandler();
        if (errorHandler != null) {
            try {
                doInvokeListener(listener, event);
            }
            catch (Throwable err) {
                // 交由错误处理器处理异常
                errorHandler.handleError(err);
            }
        }
        else {
            // 如果ApplicationListener方法执行时产生异常，则不处理，由上级处理
            doInvokeListener(listener, event);
        }
    }

    private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
        try {
            // onApplicationEvent方法就是我们平时使用ApplicationLisenter接口时要实现的方法，回调观察者
            listener.onApplicationEvent(event);
        }
        catch (ClassCastException ex) {
            String msg = ex.getMessage();
            if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
                // Possibly a lambda-defined listener which we could not resolve the generic event type for
                // -> let's suppress the exception and just log a debug message.
                Log logger = LogFactory.getLog(getClass());
                if (logger.isTraceEnabled()) {
                    logger.trace("Non-matching event type for listener: " + listener, ex);
                }
            }
            else {
                throw ex;
            }
        }
    }
}
```

从上面第47行代码可以看到这是一个for循环，for循环的对象是getApplications()方法的返回值，这里我们不关注是怎么找到对ApplicationEnvironmentPreparedEvent事件感兴趣的ApplicationListener的，重点看getApplicaitons()方法的返回值有哪些，有哪些ApplicationListener的实现类对ApplicationEnvironmentPreparedEvent事件感兴趣，下面具体分析这些监听器的实现。

#### 3.6.4.2、接收ApplicationEnvironmentPreparedEvent事件的ApplicationListener

使用idea调试时getApplications()方法的返回值(spring cloud环境)：

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200104151616.png)



按照执行顺序这些监听器的实现出处如下（注意下面文件的内容不是实际的文件内容顺序，是按照实际执行顺序列出）：

- `spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories` 共2个

  ```properties
  ## Application Listeners
  org.springframework.context.ApplicationListener=\
  org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
  org.springframework.cloud.bootstrap.LoggingSystemShutdownListener,\
  ...省略不相关的
  ```

  

- `spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories` 

  ```properties
  ## Application Listeners
  org.springframework.context.ApplicationListener=\
  org.springframework.boot.context.config.ConfigFileApplicationListener,\
  org.springframework.boot.context.config.AnsiOutputApplicationListener,\
  org.springframework.boot.context.logging.LoggingApplicationListener,\
  ...省略不相关的
  ```

- `spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories`共1个

  ```properties
  # Application Listeners
  org.springframework.context.ApplicationListener=\
  org.springframework.boot.autoconfigure.BackgroundPreinitializer
  ```

- `spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories` 

  ```properties
  ## Application Listeners
  org.springframework.context.ApplicationListener=\
  org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
  org.springframework.boot.context.config.DelegatingApplicationListener,\
  org.springframework.boot.context.FileEncodingApplicationListener,\
  ...省略不相关的
  ```

根据`getApplications()`方法的返回值看到第一个监听了`ApplicationEnvironmentPreparedEvent`事件的`ApplicationListener`就是spring cloud的`org.springframework.cloud.bootstrap.BootstrapApplicationListener`, 下面开始分析BootstrapApplicationListener`类的源码

#### 3.6.4.3、spring cloud的BootstrapApplicationListener

```java
public class BootstrapApplicationListener
    implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered {

    /**
	 * Property source name for bootstrap.
	 */
    public static final String BOOTSTRAP_PROPERTY_SOURCE_NAME = "bootstrap";

    /**
	 * The default order for this listener.
	 */
    public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 5;

    /**
	 * The name of the default properties.
	 */
    public static final String DEFAULT_PROPERTIES = "springCloudDefaultProperties";

    private int order = DEFAULT_ORDER;

    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        // 这里的environment是SpringApplication.run(xxx.class, args)时创建的StandardServletEnvironment
        // 属性源包括configurationProperties、servletConfigInitParams、servletContextInitParams、systemProperties、systemEnvironment
        ConfigurableEnvironment environment = event.getEnvironment();
        
        // 判断spring.cloud.bootstrap.enabled的属性值，若未设置则默认true
        // 这段代码的意思是是否启用了springg cloud，默认true启动
        if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class, true)) {
            // 已经设置过spring.cloud.bootstrap.enabled属性为false，代表不启用spring cloud功能，直接返回
            // 那么就不会创建id为bootstrap的父ApplicationContext
            return;
        }
        
        // 运行到这里表示启用spring cloud
        
        // don't listen to events in a bootstrap context
        // 不监听bootstrap的发布的ApplicationEnvironmentPreparedEvent事件， 防止重复创建id为bootstrap的ApplicationContext
        // 这里判断environment中是否包含名称为"bootstrap"的属性源，
        // 注意后续的bootstrapServiceContext()方法中会为spring cloud的environment添加名称为"bootstrap"的属性源，
        // 所以bootstrapServiceContext()方法中为spring cloud创建的SpringApplication在调用其run()方法时发布
        // ApplicationEnvironmentPreparedEvent事件又会进入当前类中，但是由于该bootstrap的environment(spring cloud的)已经
        // 有名称为"bootstrap"的属性源，所以不会继续往后执行。
        if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
            return;
        }
        ConfigurableApplicationContext context = null;
        // 从命令行参数、系统属性、系统环境变量解析spring.cloud.bootstrap.name的值，如果都未指定则使用默认名称bootstrap
        String configName = environment.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
        
        // 这里的event.getSpringApplication().getInitializers()返回的是SpringApplication.run()方法启动时SpringApplication的构造
        // 方法中通过SpringFacoriesLoader获取到的ApplicationContextInitializer。
        // 按照顺序如下, 括号中的jar文件名代表其对应的spring.factories文件出处：
        // org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.config.DelegatingApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.ContextIdApplicationContextInitializer  （spring-boot-2.2.1.RELEASE.jar）
        //  org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        for (ApplicationContextInitializer<?> initializer : event.getSpringApplication().getInitializers()) {
            if (initializer instanceof ParentContextApplicationContextInitializer) {
                // 不会运行到这里，上面所有列出来的类都不是ParentContextApplicationContextInitializer类型
                context = findBootstrapContext((ParentContextApplicationContextInitializer) initializer, configName);
            }
        }
        
        // 运行到这里context为null
        if (context == null) {
            // 创建id为bootstrap的父ApplicationContext
            context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);
            event.getSpringApplication()
                .addListeners(new CloseContextOnFailureApplicationListener(context));
        }

        apply(context, event.getSpringApplication(), environment);
    }
}
```

创建spring cloud的ApplicationContext

`context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);`这行代码的实现如下：

```java
private ConfigurableApplicationContext bootstrapServiceContext(
    ConfigurableEnvironment environment, final SpringApplication application, String configName) {
    // 创建spring cloud 的environment
    StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
    // 获取上面创建spring cloud 的可变属性源
    MutablePropertySources bootstrapProperties = bootstrapEnvironment.getPropertySources();
    for (PropertySource<?> source : bootstrapProperties) {
        bootstrapProperties.remove(source.getName());
    }

    // 从spring boot的environment获取"spring.cloud.bootstrap.location"属性
    String configLocation = environment.resolvePlaceholders("${spring.cloud.bootstrap.location:}");

    // 创建spring cloud的属性源的数据源，后面会把这个map加进去spring cloud的environment中
    Map<String, Object> bootstrapMap = new HashMap<>();
    // 特别注意："spring.config.name"是用于加载配置文件的名称，默认为bootstrap，
    // 在ConfigFileApplicationListener事件中会获取属性为"spring.config.name"的值作为配置文件的名称去加载配置文件
    bootstrapMap.put("spring.config.name", configName);
    bootstrapMap.put("spring.main.web-application-type", "none");
    if (StringUtils.hasText(configLocation)) {
        // 特别注意："spring.config.location"是用于加载配置文件的搜索路径，默认为空，
        // 在ConfigFileApplicationListener事件中会获取属性为"spring.config.location"的值，从该路径下去加载配置文件
        bootstrapMap.put("spring.config.location", configLocation);
    }
    // 为spring cloud的environment添加name为"bootstrap"的属性源，属性源中包括2个属性"spring.config.name"及"spring.config.location"
    bootstrapProperties.addFirst(new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));

    // 遍历spring boot的属性源
    for (PropertySource<?> source : environment.getPropertySources()) {
        // 如果是占位符的属性源就跳过
        if (source instanceof StubPropertySource) {
            continue;
        }
        // 将spring boot的属性源添加到spring cloud的environment中，
        // 注意这里是addLast
        bootstrapProperties.addLast(source);
    }

    // 创建spring cloud自身的SpringApplication了，
    // SpringApplicationBuilder的构造方法会创建SpringApplication对象（又获取ApplicationContextInitializer和ApplicationListener）
    SpringApplicationBuilder builder = new SpringApplicationBuilder()
        // 指定激活的环境，此处activeProfiles是通过系统变量"spring.profiles.active"设置的
        .profiles(environment.getActiveProfiles())
        // 关闭banner输出
        .bannerMode(Mode.OFF)
        // 设置spring cloud的环境，设置后在调用run()方法时就不会再创建了
        // 这里可以回顾下SpringApplication.getOrCreateEnvironment()方法的实现
        .environment(bootstrapEnvironment)
        // 不打印启动日志
        .registerShutdownHook(false).logStartupInfo(false)
        // 非web
        .web(WebApplicationType.NONE);
    // 返回SpringApplicationBuilder创建的SpringApplication
    final SpringApplication builderApplication = builder.application();
    // SpringApplication的构造方法中有this.mainApplicationClass = deduceMainApplicationClass();这行代码会获取mainApplicationClass
    // 所以builderApplication.getMainApplicationClass()一般不会返回为null
    if (builderApplication.getMainApplicationClass() == null) {
        // 不会进入这里
        // gh_425:
        // SpringApplication cannot deduce the MainApplicationClass here
        // if it is booted from SpringBootServletInitializer due to the
        // absense of the "main" method in stackTraces.
        // But luckily this method's second parameter "application" here
        // carries the real MainApplicationClass which has been explicitly
        // set by SpringBootServletInitializer itself already.
        builder.main(application.getMainApplicationClass());
    }
    if (environment.getPropertySources().contains("refreshArgs")) {
        // If we are doing a context refresh, really we only want to refresh the
        // Environment, and there are some toxic listeners (like the
        // LoggingApplicationListener) that affect global static state, so we need a
        // way to switch those off.
        builderApplication
            .setListeners(filterListeners(builderApplication.getListeners()));
    }
    builder.sources(BootstrapImportSelectorConfiguration.class);
    // 关键代码，创建spring cloud自身的ApplicationContext，又会把spring boot的SpringApplication.run()方法的流程全部走一遍
    final ConfigurableApplicationContext context = builder.run();
    // gh-214 using spring.application.name=bootstrap to set the context id via
    // `ContextIdApplicationContextInitializer` prevents apps from getting the actual
    // spring.application.name
    // during the bootstrap phase.
    // 设置spring cloud的ApplicationContext的id为"bootstrap"
    context.setId("bootstrap");
    // 这里的addAncestorInitializer方法的第1个参数application是spring boot的SpringApplication对象
    // 而第2个参数context是上一行代码返回的spring cloud的ApplicationContext
    addAncestorInitializer(application, context);
    // spring cloud的环境中现在有一些属性是我们不想在父ApplicationContext中看到的，所以把它移除(稍后会添加回来)
    bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
    // 合并spring cloud中name为"defaultProperties"属性源至spring boot中，注意同名属性不覆盖
    mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
    // 返回spring cloud的ApplicationContext
    return context;
}

/**
*
* @param  application   该属性是spring boot的SpringApplication对象
* @param  context       spring cloud的ApplicationContext
*/
private void addAncestorInitializer(SpringApplication application,
                                    ConfigurableApplicationContext context) {
    boolean installed = false;
    for (ApplicationContextInitializer<?> initializer : application
         .getInitializers()) {
        if (initializer instanceof AncestorInitializer) {
            installed = true;
            // New parent
            ((AncestorInitializer) initializer).setParent(context);
        }
    }
    if (!installed) {
        application.addInitializers(new AncestorInitializer(context));
    }

}

/**
* 这个方法的目的如下：
* 1、如果spring cloud中有name为"springCloudDefaultProperties"的属性源，而spring boot中没有该名称的属性源，则直接把
*    spring cloud的name为"springCloudDefaultProperties"的属性源加到spring boot的environment的末尾。
*
* 2、如果spring cloud中有name为"springCloudDefaultProperties"的属性源，spring boot中也有该名称的属性源，则把spring cloud
*    的属性源合并到spring boot的environment中，如果在spring cloud中的属性在spring boot中不存在则追加到spring boot中，特别注意
*    如果spring cloud中的属性在spring boot中也存在，则不会覆盖，也就是说spring boot和spring cloud拥有相同key的属性时，spring boot的
*    配置文件中的属性优先级高，不会合并该属性。
* 
* @param environment 它是spring boot的environment
* @param bootstrap 它是spring cloud的environment
*/
private void mergeDefaultProperties(MutablePropertySources environment,
                                    MutablePropertySources bootstrap) {
    String name = DEFAULT_PROPERTIES;
    // 判断spring cloud中是否有name为"springCloudDefaultProperties"的属性源
    if (bootstrap.contains(name)) {
        // 获得spring cloud中name为"springCloudDefaultProperties"的属性源
        PropertySource<?> source = bootstrap.get(name);
        // 判断spring boot中是否有name为"springCloudDefaultProperties"的属性源
        if (!environment.contains(name)) {
            // 如果spring cloud中有name为"springCloudDefaultProperties"的属性源，而spring boot中没有该名称的属性源，则直接把
            // spring cloud的name为"springCloudDefaultProperties"的属性源加到spring boot的environment的末尾
            environment.addLast(source);
        }
        // spring cloud中有name为"springCloudDefaultProperties"的属性源，spring boot中也有该名称的属性源
        else {
            // 获得spring boot中name为"springCloudDefaultProperties"的属性源
            PropertySource<?> target = environment.get(name);
            // 判断spring boot和spring cloud的name为"springCloudDefaultProperties"的属性源的类型是否是MapPropertySource类型
            if (target instanceof MapPropertySource && target != source
                && source instanceof MapPropertySource) {
                // 获得spring boot中name为"springCloudDefaultProperties"的属性源的底层数据源
                Map<String, Object> targetMap = ((MapPropertySource) target).getSource();
                // 获得spring cloud中name为"springCloudDefaultProperties"的属性源的底层数据源
                Map<String, Object> map = ((MapPropertySource) source).getSource();

                // 遍历spring cloud中name为"springCloudDefaultProperties"的属性源的底层数据源
                for (String key : map.keySet()) {
                    // 判断spring cloud中的属性在spring boot是否存在，若spring cloud中的属性在spring boot中不存在则追加到spring boot中 
                    if (!target.containsProperty(key)) {
                        targetMap.put(key, map.get(key));
                    }

                    // 若spring cloud中的属性在spring boot中存在则忽略，spring cloud不覆盖spring boot的同名属性
                }
            }
        }
    }
    mergeAdditionalPropertySources(environment, bootstrap);
}
```







```
org.springframework.cloud.bootstrap.BootstrapApplicationListener
	org.springframework.cloud.bootstrap.LoggingSystemShutdownListener
	org.springframework.boot.context.config.ConfigFileApplicationListener(SmartApplicationListener)
    	org.springframework.boot.env.EnvironmentPostProcessor=
    		org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
    		org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
    		org.springframework.cloud.client.HostInfoEnvironmentPostProcessor
            org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
            org.springframework.boot.context.config.ConfigFileApplicationListener
            	# PropertySource Loaders
                org.springframework.boot.env.PropertySourceLoader=\
                    org.springframework.boot.env.PropertiesPropertySourceLoader,\
                    org.springframework.boot.env.YamlPropertySourceLoader
            org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor
```



#### 3.6.4.4、ConfigFileApplicationListener

该类的作用是加载配置文件

```java
public class ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationEnvironmentPreparedEvent) {
            onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
        }
        if (event instanceof ApplicationPreparedEvent) {
            onApplicationPreparedEvent(event);
        }
    }

    private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
        List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
        // 当前类本身实现了EnvironmentPostProcessor接口，把自己加到postProcessors集合中，
        postProcessors.add(this);
        
         // 进行排序，排序后此时的postProcessors按照执行顺序如下：
        // org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor  （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor   （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.cloud.client.HostInfoEnvironmentPostProcessor          （spring-cloud-commons-2.2.0.RELEASE.jar）
        // org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor   （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.config.ConfigFileApplicationListener      （上一行把自己添加进来的，不是来源于spring.factories文件中）
        // org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor        （spring-boot-2.2.1.RELEASE.jar）
        AnnotationAwareOrderComparator.sort(postProcessors);
        // 遍历处理每一个EnvironmentPostProcessor
        for (EnvironmentPostProcessor postProcessor : postProcessors) {
            postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
        }
    }

    private void onApplicationPreparedEvent(ApplicationEvent event) {
        this.logger.switchTo(ConfigFileApplicationListener.class);
        addPostProcessors(((ApplicationPreparedEvent) event).getApplicationContext());
    }

    List<EnvironmentPostProcessor> loadPostProcessors() {
        // 返回的EnvironmentPostProcessor按照执行顺序如下：
        // 括号中的代表该自动化配置的所在的spring.factories文件出处
        // org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor  （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor   （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.cloud.client.HostInfoEnvironmentPostProcessor          （spring-cloud-commons-2.2.0.RELEASE.jar）
        // org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor   （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor        （spring-boot-2.2.1.RELEASE.jar）
        return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class, getClass().getClassLoader());
    }
    
    // 当前类实现了EnvironmentPostProcessor接口
    @Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		addPropertySources(environment, application.getResourceLoader());
	}
}
```

- `SystemEnvironmentPropertySourceEnvironmentPostProcessor`

  ``` java
  public class SystemEnvironmentPropertySourceEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
  
      @Override
      public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
          // sourceName的值为"systemEnvironment"
          String sourceName = StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME;
          // 获取name为"systemEnvironment"的系统环境变量属性源
          PropertySource<?> propertySource = environment.getPropertySources().get(sourceName);
          // 一般都不会为null
          if (propertySource != null) {
              // 会执行到这里，调用当前类的replacePropertySource方法
              replacePropertySource(environment, sourceName, propertySource);
          }
      }
  
      /**
      *
      * @param environment spring boot/cloud的环境对象，servlet环境是StandardServletEnvironment
      * @param sourceName  值为"systemEnvironment"
      * @parm propertySource  名称为"systemEnvironment"的系统环境变量属性源
      */
      private void replacePropertySource(ConfigurableEnvironment environment, String sourceName,
                                         PropertySource<?> propertySource) {
          // 获取所有系统环境变量
          Map<String, Object> originalSource = (Map<String, Object>) propertySource.getSource();
          SystemEnvironmentPropertySource source = new OriginAwareSystemEnvironmentPropertySource(sourceName,
                                                                                                  originalSource);
          // 把name为"systemEnvironment"的系统环境变量属性源替换为OriginAwareSystemEnvironmentPropertySource
          // 有什么作用？？目前还不清楚为什么要这样做。
          environment.getPropertySources().replace(sourceName, source);
      }
  
  }
  ```

  

- `SpringApplicationJsonEnvironmentPostProcessor`

  ```java
  /**
  *
  * 找到environment中Key为"spring.application.json"的属性，将其值解析为Map对象，作为一个属性源，
  * 向environment中添加1个name为"spring.application.json"的JsonPropertySource属性源，数据源是之前描述的解析得到的Map对象
  */
  public class SpringApplicationJsonEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
  
      public static final String SPRING_APPLICATION_JSON_PROPERTY = "spring.application.json";
  
  
      public static final String SPRING_APPLICATION_JSON_ENVIRONMENT_VARIABLE = "SPRING_APPLICATION_JSON";
  
  
      @Override
      public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
          // 获取可变属性源，包括springApplicationCommandLineArgs、configurationProperties、bootstrap、systemProperties、systemEnvironment属性源
          MutablePropertySources propertySources = environment.getPropertySources();
          propertySources.stream()
              //  获取"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性的实际值对应的JsonPropertyValue对象
              // 如果当前属性源不存在该属性则返回null
              // 返回的数据形式如[null, JsonPropertyValue, null, JsonPropertyValue, JsonPropertyValue, null...]
              .map(JsonPropertyValue::get)
              // 找出第一个非null的JsonPropertyValue
              .filter(Objects::nonNull).findFirst()
              // 如果存在"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性的实际值对应的JsonPropertyValue对象则进行处理
              .ifPresent((v) -> processJson(environment, v));
      }
      
      // 处理spring boot/cloud环境中"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性的实际值对应的JsonPropertyValue对象
      private void processJson(ConfigurableEnvironment environment, JsonPropertyValue propertyValue) {
          // 根据classpath类路径是否存在com.fasterxml.jackson.databind.ObjectMapper、com.google.gson.Gson、org.yaml.snakeyaml.Yaml
          // 类返回对应的解析器JacksonJsonParser、GsonJsonParser、YamlJsonParser、BasicJsonParser
  		JsonParser parser = JsonParserFactory.getJsonParser();
           // propertyValue.getJson()的值为environment中"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性的值
          // 使用jackson、gson、yaml将字符串解析为Map类型对象
  		Map<String, Object> map = parser.parseMap(propertyValue.getJson());
  		if (!map.isEmpty()) {
               // 添加name为"spring.application.json"的属性源 
  			addJsonPropertySource(environment, new JsonPropertySource(propertyValue, flatten(map)));
  		}
  	}
  
      private static class JsonPropertyValue {
  		// 实际值为 ["spring.application.json", "SPRING_APPLICATION_JSON"]
          private static final String[] CANDIDATES = { SPRING_APPLICATION_JSON_PROPERTY,
                                                      SPRING_APPLICATION_JSON_ENVIRONMENT_VARIABLE };
          
          private final PropertySource<?> propertySource;
  
  		private final String propertyName;
  
  		private final String json;
  
          /**
          * @param propertySource "spring.application.json" 或 "SPRING_APPLICATION_JSON"属性所在的属性源
          * @param propertyName 表示根据哪个属性名找到的属性值的，值为字符串"spring.application.json" 或 "SPRING_APPLICATION_JSON"其中之一
          * @param json 在envorpnment中"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性对应的值
          */
  		JsonPropertyValue(PropertySource<?> propertySource, String propertyName, String json) {
  			this.propertySource = propertySource;
  			this.propertyName = propertyName;
  			this.json = json;
  		}
  
          /**
          *
          * @param propertySource  属性源
          */
          static JsonPropertyValue get(PropertySource<?> propertySource) {
              for (String candidate : CANDIDATES) {
                  // 获取"spring.application.json" 或 "SPRING_APPLICATION_JSON"属性的实际值
                  Object value = propertySource.getProperty(candidate);
                  // 判断属性的值是否是字符串并且有内容
                  if (value instanceof String && StringUtils.hasLength((String) value)) {
                      // 返回当前类的实例
                      return new JsonPropertyValue(propertySource, candidate, (String) value);
                  }
              }
              return null;
          }
      }
      
      private static class JsonPropertySource extends MapPropertySource implements OriginLookup<String> {
  
  		private final JsonPropertyValue propertyValue;
  
  		JsonPropertySource(JsonPropertyValue propertyValue, Map<String, Object> source) {
               // 属性源的name为"spring.application.json"
  			super(SPRING_APPLICATION_JSON_PROPERTY, source);
  			this.propertyValue = propertyValue;
  		}
  
  		@Override
  		public Origin getOrigin(String key) {
  			return this.propertyValue.getOrigin();
  		}
  
  	}
      
  }
  ```

  

- `HostInfoEnvironmentPostProcessor`

  ```java
  public class HostInfoEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
      @Override
      public void postProcessEnvironment(ConfigurableEnvironment environment,
                                         SpringApplication application) {
          // 找到第1个非回环地址(非127.xxx.xxx.xxx)信息
          InetUtils.HostInfo hostInfo = getFirstNonLoopbackHostInfo(environment);
          LinkedHashMap<String, Object> map = new LinkedHashMap<>();
          // 计算机名，例如我的电脑是calebzhao，win10系统在桌面图标“此电脑”右键菜单点击属性即可看到计算机名
          map.put("spring.cloud.client.hostname", hostInfo.getHostname());
          // 电脑ip地址，不是本地回环地址，一般个人计算机是192.168.xxx.xxx
          map.put("spring.cloud.client.ip-address", hostInfo.getIpAddress());
          MapPropertySource propertySource = new MapPropertySource(
              "springCloudClientHostInfo", map);
          // 添加了一个名称为springCloudClientHostInfo的属性源
          environment.getPropertySources().addLast(propertySource);
      }
  
      private HostInfo getFirstNonLoopbackHostInfo(ConfigurableEnvironment environment) {
          InetUtilsProperties target = new InetUtilsProperties();
          ConfigurationPropertySources.attach(environment);
          Binder.get(environment).bind(InetUtilsProperties.PREFIX,
                                       Bindable.ofInstance(target));
          try (InetUtils utils = new InetUtils(target)) {
              return utils.findFirstNonLoopbackHostInfo();
          }
      }
  }
  ```

  

- `ConfigFileApplicationListener`

  默认从`["file:./config/", "file:./", "classpath:/config/", "classpath:/"]`目录下加载配置文件

  ```java
  public class ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {
  
      private static final String DEFAULT_PROPERTIES = "defaultProperties";
  
      public static final String ACTIVE_PROFILES_PROPERTY = "spring.profiles.active";
  
      public static final String INCLUDE_PROFILES_PROPERTY = "spring.profiles.include";
  
      static {
          Set<String> filteredProperties = new HashSet<>();
          filteredProperties.add("spring.profiles.active");
          filteredProperties.add("spring.profiles.include");
          LOAD_FILTERED_PROPERTY = Collections.unmodifiableSet(filteredProperties);
      }
  
      // 入口
      @Override
      public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
          addPropertySources(environment, application.getResourceLoader());
      }
  
      protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
          // 在systemEnvironment属性源之后添加能够获取随机数的属性源，属性源的名称为"random"
          RandomValuePropertySource.addToEnvironment(environment);
          // 加载
          new Loader(environment, resourceLoader).load();
      }
  
  
      private class Loader {
          private final ConfigurableEnvironment environment;
  
          private final PropertySourcesPlaceholdersResolver placeholdersResolver;
  
          private final ResourceLoader resourceLoader;
  
          private final List<PropertySourceLoader> propertySourceLoaders;
  
          private Deque<Profile> profiles;
  
          Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
              this.environment = environment;
              this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
              this.resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader();
  
              // 这里非常关键，到spring.factories中找key为org.springframework.boot.env.PropertySourceLoader的属性源加载器,
              // 括号中的内容代表该PropertySourceLoader所在的spring.factories文件所在的jar文件，按照执行顺序找到的实现如下：
              // org.springframework.boot.env.PropertiesPropertySourceLoader  （spring-boot-2.2.1.RELEASE.jar）
              // org.springframework.boot.env.YamlPropertySourceLoader       （spring-boot-2.2.1.RELEASE.jar）
              this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
                                                                               getClass().getClassLoader());
          }
  
          // 实际处理逻辑：加载项目下的application.yml，，默认从["file:./config/", "file:./", "classpath:/config/", "classpath:/"]加载配置文件
          void load() {
              // environment包括springApplicationCommandLineArgs、configurationProperties、【bootstrap】、systemProperties、systemEnvironment属性源
              FilteredPropertySource.apply(this.environment, 
                                           DEFAULT_PROPERTIES,  // defaultProperties
                                           LOAD_FILTERED_PROPERTY, // ["spring.profiles.active", "spring.profiles.include"]
                                           (defaultProperties) -> {
  
                                               this.profiles = new LinkedList<>();
                                               this.processedProfiles = new LinkedList<>();
                                               this.activatedProfiles = false;
                                               this.loaded = new LinkedHashMap<>();
                                               // 1.把环境中的profiles取出来，默认会增加1个为null的profile
                                               // 如果用户如果自己通过命令行参数、jvm系统属性、系统环境变量指定了"spring.profiles.active"
                                               // 或"spring.profiles.include"属性值则不使用默认的profile，
                                               // 否则表明用户没有明确指定启用哪些profile，那么就使用spring默认的profile，
                                               // 用户设置可以覆盖spring默认设置
                                               initializeProfiles();
                                                // 2.循环处理profiles，查找文件位置然后去加载文件
                                               while (!this.profiles.isEmpty()) {
                                                   Profile profile = this.profiles.poll();
                                                   // 判断是否是默认profile
                                                   if (isDefaultProfile(profile)) {
                                                       addProfileToEnvironment(profile.getName());
                                                   }
  
                                                   // 重点：加载该环境的配置文件
                                                   load(profile, this::getPositiveProfileFilter,
                                                        addToLoaded(MutablePropertySources::addLast, false));
                                                   this.processedProfiles.add(profile);
                                               }
                                               load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
                                               addLoadedPropertySources();
                                               applyActiveProfiles(defaultProperties);
                                           }
                                          );
          }
  
          private void initializeProfiles() {
              // The default profile for these purposes is represented as null. We add it
              // first so that it is processed first and has lowest priority.
              this.profiles.add(null);
              //  ACTIVE_PROFILES_PROPERTY常量的值为"spring.profiles.active"
              // 从environment中取key为"spring.profiles.active"的值，若未找到返回空集合
              Set<Profile> activatedViaProperty = getProfilesFromProperty(ACTIVE_PROFILES_PROPERTY);
              // INCLUDE_PROFILES_PROPERTY常量的值为"spring.profiles.include"
              // 从environment中取key为"spring.profiles.include"的值，若未找到返回空集合
              Set<Profile> includedViaProperty = getProfilesFromProperty(INCLUDE_PROFILES_PROPERTY);
              // 从environment中找到profile名称不在activatedViaProperty及includedViaProperty集合内的已激活的profile的名称
              List<Profile> otherActiveProfiles = getOtherActiveProfiles(activatedViaProperty, includedViaProperty);
              this.profiles.addAll(otherActiveProfiles);
              // Any pre-existing active profiles set via property sources (e.g.
              // System properties) take precedence over those added in config files.
              this.profiles.addAll(includedViaProperty);
              addActiveProfiles(activatedViaProperty);
              // 如果用户如果自己通过命令行参数、jvm系统属性、系统环境变量指定了"spring.profiles.active"或"spring.profiles.include"属性值
              // 则不使用默认的profile，否则表明用户没有明确指定启用哪些profile，那么就使用spring默认的profile，用户设置可以覆盖spring默认设置
              if (this.profiles.size() == 1) { // profiles集合中只包含1个null元素
                  // 获取默认profile的名称，this.environment.getDefaultProfiles()方法默认返回字符串"default"
                  for (String defaultProfileName : this.environment.getDefaultProfiles()) {
                      // 创建name为"default"的Profile对象
                      Profile defaultProfile = new Profile(defaultProfileName, true);
                      // 将默认profile加到profiles中，此时profiles集合变成 [ null, 名称为"default"的Profile对象 ]
                      this.profiles.add(defaultProfile);
                  }
              }
          }
  
          private Set<Profile> getProfilesFromProperty(String profilesProperty) {
              if (!this.environment.containsProperty(profilesProperty)) {
                  return Collections.emptySet();
              }
              Binder binder = Binder.get(this.environment);
              Set<Profile> profiles = getProfiles(binder, profilesProperty);
              return new LinkedHashSet<>(profiles);
          }
  
          private void load(Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
              // 确定从哪些位置搜索配置文件，默认从["file:./config/", "file:./", "classpath:/config/", "classpath:/"]搜索配置文件
              getSearchLocations() 
                  // 遍历每个搜索路径
                  .forEach((location) -> {
                      // 判断搜索路径是否是以"/"结尾的，如果是说明指定的目录，否则说明指定的搜索路径精确到具体文件名了
                      boolean isFolder = location.endsWith("/");
                      // NO_SEARCH_NAMES集合默认只有1个null元素
                      // 如果location路径是文件夹，在用户没有明确指定"spring.config.name"属性的情况下，
                      // 对于spring boot返回的是["application"]，对于spring cloud返回的是["bootstrap"]
                      Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
                      // 重点：根据names循环加载可能的配置文件
                      names.forEach((name) -> load(location, name, profile, filterFactory, consumer));
                  });
          }
  
          private void load(String location, String name, Profile profile, DocumentFilterFactory filterFactory,
                            DocumentConsumer consumer) {
              // 判断name是否有值
              if (!StringUtils.hasText(name)) {
                  for (PropertySourceLoader loader : this.propertySourceLoaders) {
                      if (canLoadFileExtension(loader, location)) {
                          load(loader, location, profile, filterFactory.getDocumentFilter(profile), consumer);
                          return;
                      }
                  }
                  throw new IllegalStateException("File extension of config file location '" + location
                                                  + "' is not known to any PropertySourceLoader. If the location is meant to reference "
                                                  + "a directory, it must end in '/'");
              }
              Set<String> processed = new HashSet<>();
  
              // this.propertySourceLoaders的值是在Loader类的构造方法中通过
              // SpringFactoriesLoader.loadFactories(PropertySourceLoader.class, getClass().getClassLoader())初始化的。
  
              // this.propertySourceLoaders集合属性值按照顺序包括:
              // org.springframework.boot.env.PropertiesPropertySourceLoader
              // org.springframework.boot.env.YamlPropertySourceLoader
              for (PropertySourceLoader loader : this.propertySourceLoaders) {
                  // loader.getFileExtensions()方法返回值分如下2种情况：
                  // 对于org.springframework.boot.env.PropertiesPropertySourceLoader返回 ["properties", "xml"]
                  // 对于org.springframework.boot.env.YamlPropertySourceLoader返回 ["yml", "yaml"]
                  for (String fileExtension : loader.getFileExtensions()) {
                      if (processed.add(fileExtension)) {
                          // 核心方法：加载spring boot、spring cloud的配置文件
                          loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory,
                                               consumer);
                      }
                  }
              }
          }
  
          /**
          *
          * @param loader 属性加载器，PropertiesPropertySourceLoader或YamlPropertySourceLoader
          * @param prefix 不包括扩展名的路径，例如"file:./config/bootstrap"、"classpath:./config/application"、"classpath:/application"等
          * @param fileExtension 文件扩展名，例如".yml"、".yaml"、".properties"、".xml"
          * @param profile 启用的环境 
          *
          */
          private void loadForFileExtension(PropertySourceLoader loader, String prefix, String fileExtension,
                                            Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
              DocumentFilter defaultFilter = filterFactory.getDocumentFilter(null);
              DocumentFilter profileFilter = filterFactory.getDocumentFilter(profile);
              if (profile != null) {
                  // 加载特定环境的配置文件
                  // 例如"file:./config/bootstrap-default.yml" 、"classpath:/application-default.yml"
                  String profileSpecificFile = prefix + "-" + profile + fileExtension;
                  load(loader, profileSpecificFile, profile, defaultFilter, consumer);
                  load(loader, profileSpecificFile, profile, profileFilter, consumer);
                  // Try profile specific sections in files we've already processed
                  for (Profile processedProfile : this.processedProfiles) {
                      if (processedProfile != null) {
                          String previouslyLoaded = prefix + "-" + processedProfile + fileExtension;
                          load(loader, previouslyLoaded, profile, profileFilter, consumer);
                      }
                  }
              }
              // 加载常规配置文件
              // 例如"file:./config/bootstrap.yml" 、"classpath:/application.yml"
              load(loader, prefix + fileExtension, profile, profileFilter, consumer);
          }
          
          private void load(PropertySourceLoader loader, String location, Profile profile, DocumentFilter filter,
  				DocumentConsumer consumer) {
  			try {
                    // 真正加载配置文件了
  				Resource resource = this.resourceLoader.getResource(location);
  				if (resource == null || !resource.exists()) {
  					if (this.logger.isTraceEnabled()) {
  						StringBuilder description = getDescription("Skipped missing config ", location, resource,
  								profile);
  						this.logger.trace(description);
  					}
  					return;
  				}
  				if (!StringUtils.hasText(StringUtils.getFilenameExtension(resource.getFilename()))) {
  					if (this.logger.isTraceEnabled()) {
  						StringBuilder description = getDescription("Skipped empty config extension ", location,
  								resource, profile);
  						this.logger.trace(description);
  					}
  					return;
  				}
  				String name = "applicationConfig: [" + location + "]";
  				List<Document> documents = loadDocuments(loader, name, resource);
  				if (CollectionUtils.isEmpty(documents)) {
  					if (this.logger.isTraceEnabled()) {
  						StringBuilder description = getDescription("Skipped unloaded config ", location, resource,
  								profile);
  						this.logger.trace(description);
  					}
  					return;
  				}
  				List<Document> loaded = new ArrayList<>();
  				for (Document document : documents) {
  					if (filter.match(document)) {
  						addActiveProfiles(document.getActiveProfiles());
  						addIncludedProfiles(document.getIncludeProfiles());
  						loaded.add(document);
  					}
  				}
  				Collections.reverse(loaded);
  				if (!loaded.isEmpty()) {
  					loaded.forEach((document) -> consumer.accept(profile, document));
  					if (this.logger.isDebugEnabled()) {
  						StringBuilder description = getDescription("Loaded config file ", location, resource, profile);
  						this.logger.debug(description);
  					}
  				}
  			}
  			catch (Exception ex) {
  				throw new IllegalStateException("Failed to load property source from location '" + location + "'", ex);
  			}
  		}
  
          // 确定从哪些位置搜索配置文件
          private Set<String> getSearchLocations() {
              // CONFIG_LOCATION_PROPERTY常量值为"spring.config.location"
              // 这里的意图是判断用户是否通过命令行参数、系统属性、系统环境变量指定了"spring.config.location"属性值
              // 如果指定了"spring.config.location"属性值则使用用户指定的配置文件路径
              if (this.environment.containsProperty(CONFIG_LOCATION_PROPERTY)) {
                  return getSearchLocations(CONFIG_LOCATION_PROPERTY);
              }
  
              // CONFIG_ADDITIONAL_LOCATION_PROPERTY常量值为"spring.config.additional-location"
              // 这里的意图是判断命令行参数、系统属性、系统环境变量是否指定了"spring.config.additional-location"属性值
              // 如果指定了说明除了要解析默认的配置文件路径外，还要额外解析指定位置的配置文件，这就是additional-location所表达的意思
              Set<String> locations = getSearchLocations(CONFIG_ADDITIONAL_LOCATION_PROPERTY);
  
              // DEFAULT_SEARCH_LOCATIONS常量值为"classpath:/,classpath:/config/,file:./,file:./config/"
              locations.addAll(
                  // ConfigFileApplicationListener.this.searchLocations默认为null
                  // asResolvedSet返回值为集合["file:./config/", "file:./", "classpath:/config/", "classpath:/"]
                  asResolvedSet(ConfigFileApplicationListener.this.searchLocations, DEFAULT_SEARCH_LOCATIONS));
              return locations;
          }
  
          private Set<String> getSearchLocations(String propertyName) {
              Set<String> locations = new LinkedHashSet<>();
              // 判断environment环境中是否包含指定属性
              if (this.environment.containsProperty(propertyName)) {
                  for (String path : asResolvedSet(this.environment.getProperty(propertyName), null)) {
                      if (!path.contains("$")) {
                          path = StringUtils.cleanPath(path);
                          if (!ResourceUtils.isUrl(path)) {
                              // ResourceUtils.FILE_URL_PREFIX常量值为"file:"
                              path = ResourceUtils.FILE_URL_PREFIX + path;
                          }
                      }
                      locations.add(path);
                  }
              }
              return locations;
          }
  
          // 确定配置文件的名称
          private Set<String> getSearchNames() {
              // CONFIG_NAME_PROPERTY常量值为"spring.config.name"
              // 这里的意图是判断用户是否通过命令行参数、系统属性、系统环境变量指定了"spring.config.name"属性
  
              // 特别注意spring cloud的BootstrapApplicationListener中的bootstrapServiceContext()方法中，
              // 创建的environment添加了1个名称为bootstrap的属性源，该属性源中就指定了"spring.config.name"属性值为"bootstrap"
              if (this.environment.containsProperty(CONFIG_NAME_PROPERTY)) {
                  // 用户指定了"spring.config.name"属性，获取属性值
                  String property = this.environment.getProperty(CONFIG_NAME_PROPERTY);
                  // 对属性值以逗号分隔，返回倒序集合，例如用户通过命令行参数指定属性-D"spring.config.name=application.yml,config.yml"
                  // 则返回["config.yml", "application.yml"]
                  return asResolvedSet(property, null);
              }
  
              // DEFAULT_NAMES常量值为"application", ConfigFileApplicationListener.this.names属性默认值为null
              // 运行到这里说明用户没有指定"spring.config.name"属性，那么使用默认配置文件名称"application"
              return asResolvedSet(ConfigFileApplicationListener.this.names, DEFAULT_NAMES);
          }
  
          /**
          * 如果value为空则使用后备值fallback，否则从environment中解析value中的占位符，然后按逗号分隔并反转
          * 例如：value为null, fallback为"classpath:/,classpath:/config/,file:./,file:./config/"，则返回
          * 集合["file:./config/", "file:./", "classpath:/config/", "classpath:/"]
          */
          private Set<String> asResolvedSet(String value, String fallback) {
              // 以逗号分隔的字符串分割转集合
              List<String> list = Arrays.asList(StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(
                  (value != null) ? this.environment.resolvePlaceholders(value) : fallback)));
              // 对集合反转
              Collections.reverse(list);
              return new LinkedHashSet<>(list);
          }
      }
  }
  ```

  `FilteredPropertySource`源码

  ```java
  package org.springframework.boot.context.config;
  
  import java.util.Set;
  import java.util.function.Consumer;
  
  import org.springframework.core.env.ConfigurableEnvironment;
  import org.springframework.core.env.MutablePropertySources;
  import org.springframework.core.env.PropertySource;
  
  /**
   * Internal {@link PropertySource} implementation used by
   * {@link ConfigFileApplicationListener} to filter out properties for specific operations.
   *
   * @author Phillip Webb
   */
  class FilteredPropertySource extends PropertySource<PropertySource<?>> {
  
      private final Set<String> filteredProperties;
  
      FilteredPropertySource(PropertySource<?> original, Set<String> filteredProperties) {
          super(original.getName(), original);
          this.filteredProperties = filteredProperties;
      }
  
      @Override
      public Object getProperty(String name) {
          if (this.filteredProperties.contains(name)) {
              return null;
          }
          return getSource().getProperty(name);
      }
  
      /**
      *
      * @parma environment spring boot/cloud的环境对象，包含了springApplicationCommandLineArgs、
      *                     configurationProperties、【bootstrap】、systemProperties、systemEnvironment属性源
      *                     
      * @param propertySourceName 字符串"defaultProperties"
      * @parm  filteredProperties 集合["spring.profiles.active", "spring.profiles.include"]
      * @param operation 消费者
      * 
      */
      static void apply(ConfigurableEnvironment environment, String propertySourceName, Set<String> filteredProperties,
                        Consumer<PropertySource<?>> operation) {
          MutablePropertySources propertySources = environment.getPropertySources();
          // 获取name为"defaultProperties"的属性源
          PropertySource<?> original = propertySources.get(propertySourceName);
          if (original == null) {
              operation.accept(null);
              return;
          }
          propertySources.replace(propertySourceName, new FilteredPropertySource(original, filteredProperties));
          try {
              operation.accept(original);
          }
          finally {
              propertySources.replace(propertySourceName, original);
          }
      }
  
  }
  
  ```

  

- `DebugAgentEnvironmentPostProcessor`

### 3.6.5、绑定ConfigurableEnvironment到当前的SpringApplication实例中

```java
bindToSpringApplication(environment);
```

bindToSpringApplication()方法的具体源码如下：

```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
    try {
        Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
    }
    catch (Exception ex) {
        throw new IllegalStateException("Cannot bind to SpringApplication", ex);
    }
}
```



## 3.7、configureIgnoreBeanInfo(environment)



```java
private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
    // 获取jvm系统属性spring.beaninfo.ignore
    if (System.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
        // 如果jvm系统属性中不存在key为spring.beaninfo.ignore的系统属性，则从ConfigurableEnvironment中获取它的值
        // 如果ConfigurableEnvironment中也没有key为spring.beaninfo.ignore的属性，则使用默认值true
        // 如果从environment获取到了说明则属性可能来源于系统环境变量、命令行参数、代码设置
        Boolean ignore = environment.getProperty("spring.beaninfo.ignore", Boolean.class, Boolean.TRUE);
        // 设置到jvm系统属性中
        System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME, ignore.toString());
    }
}
```

该方法好像不太重要，不深究它。

## 3.8、创建并打印Banner

```java
Banner printedBanner = printBanner(environment);
```

这是用来打印 Banner 的处理类，这个没什么好说的。



## 3.9、创建ApplicationContext

```java
context = createApplicationContext();
```

### 3.9.1、ApplicationContext的创建

来看下 `createApplicationContext()` 方法的源码：

```java
public class SpringApplication {

    public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";

    public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

    public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

    // ...省略代码

    // 该属性值是可选的，指定使用哪个类创建ConfigurableApplicationContext，如果不指定则根据应用类型创建相应的ApplicationContext
    private Class<? extends ConfigurableApplicationContext> applicationContextClass;

    // ...省略代码

    /**
     * 用于创建ApplicationContext的策略方法。 
     * 默认情况下，此方法将使用任何明确设置的应用程序上下文或应用程序上下文类，然后再降级为使用合适的默认值。
     */
    protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                switch (this.webApplicationType) {
                    case SERVLET:
                        // servlet应用
                        contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                        break;
                    case REACTIVE:
                        // reactive应用
                        contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                        break;
                    default:
                        // 标准应用
                        contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
                }
            }
            catch (ClassNotFoundException ex) {
                throw new IllegalStateException(
                    "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
            }
        }
        // 创建ConfigurableApplicationContext实现类的实例
        return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
}
```

其实就是如果指定了applicationContextClass则使用指定的applicationContextClass创建ConfigurableApplicationContext，如果没有明确设置applicationContextClass，则根据不同的应用类型初始化不同的上下文应用类，可以看到createApplicationContext()方法返回了ConfigurableApplicationContext。

那么这个ConfigurableApplicationContext又是什么？

查看ConfigurableApplicationContext的源码，可以发现它是个接口，又继承了ApplicationContext接口，下面先看ApplicationContext接口的源码。

### 3.9.2、ApplicationContext接口

```java
/**
* 这是一个为应用程序提供配置的中央化的接口，在应用程序运行时，它是只读的，但是如果该接口的实现支持的话那么它的配置
* 是可以被重新加载的，即ApplicationContext的实现可以让配置变成可写的（非只读），在需要的时候可以重新加载配置，让配置发生变化。
*
* 一个ApplicationContext提供了如下的功能：
* 1、用于访问应用程序组件的bean工厂方法，该能力继承于org.springframework.beans.factory.ListableBeanFactory接口
* 2、以通用方式加载文件资源的能力，该能力继承于org.springframework.core.io.ResourceLoader接口
* 3、向注册的监听器发布事件的能力，该能力继承于ApplicationEventPublisher接口
* 4、解析消息的能力，支持i18n国际化，该能力继承于MessageSource接口
* 5、可以继承父上下文，在子上下文中的定义总是具有更高的优先级（其实就是父子ApplicationContext中都有相同的bean时优先使用子上下
*    文中定义的bean）,这也就意味着可以在整个web应用中使用一个单一的父上下文，而每个servlet可以有它自己的子上下文，这些子上下文
*    之间是互相独立的，而他们拥有相同的父上下文
*/
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
MessageSource, ApplicationEventPublisher, ResourcePatternResolver {

    // 返回当前应用程序上下文的唯一id，如果没有可能返回null        
    @Nullable
    String getId();

    // 返回此上下文所属的已部署应用的名称        
    String getApplicationName();

    // 为这个上下文返回一个友好可读的名称        
    String getDisplayName();

    // 返回这个上下文第一次被加载的时间戳（毫秒）        
    long getStartupDate();

    // 返回这个上下文的父上下文        
    @Nullable
    ApplicationContext getParent();

    // 为这个上下文公开AutowireCapableBeanFactory功能。
    // 除非用于初始化位于应用程序上下文之外的bean实例，并将Spring bean的生命周期(全部或部分)应用于这些实例，
    // 应用程序通常不会使用到此功能，        
    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
}
```



### 3.9.3、ConfigurableApplicationContext

```java
/**
* 这个接口是一个SPI接口，绝大多数应用(即便不是全部)程序上下文都实现了该接口，该接口除了有继承ApplicationContext接口中的
* 方法外，它还提供了用于配置应用程序上下文的功能。
*
* 与配置和与生命周期相关的方法被封装在该接口中而不是父接口ApplicationContext中，这是为了避免开发人员开发的应用可以通过
* ApplicationContext直接访问到与配置和与生命周期相关的方法，这些与配置和与生命周期相关的方法应该只能被与启动和关闭应用相关
* 的代码所使用。
*
* Lifecycle 接口则是负责对 context 的生命周期进行管理，提供了 start() 和 stop() 以及 isRunning() 方法。
*
* Closeable 接口是JDK提供的接口，用于关闭组件，释放资源。
*/
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    // 应用上下文配置时，这些符号用于分割多个配置路径
    String CONFIG_LOCATION_DELIMITERS = ",; \t\n";

    // 工厂中ConversionService bean的名称。如果没有提供，则使用默认的转换规则。
    // 参见org.springframework.core.convert.ConversionService
    String CONVERSION_SERVICE_BEAN_NAME = "conversionService";


    // LoadTimeWaver类所对应的Bean在容器中的名字。如果提供了该实例，上下文会使用临时的ClassLoader
    // 这样，LoadTimeWaver就可以使用bean确切的类型了
    //参见org.springframework.instrument.classloading.LoadTimeWeaver
    String LOAD_TIME_WEAVER_BEAN_NAME = "loadTimeWeaver";

    // Environment类在容器中的bean名称
    String ENVIRONMENT_BEAN_NAME = "environment";

    // System系统变量在容器中对应的Bean的名字，参见java.lang.System#getProperties()
    String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";

    // System 环境变量在容器中对应的Bean的名字，参见java.lang.System#getenv()
    String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";

    // 用于关闭应用时钩子线程的名称，默认SpringContextShutdownHook，参见registerShutdownHook()方法
    String SHUTDOWN_HOOK_THREAD_NAME = "SpringContextShutdownHook";

    // 为应用上下文设置一个唯一的id
    void setId(String id);

    // 为该上下文设置父上下文。
    // 需要注意的是，父上下文一经设定就不应该修改。并且一般不会在构造方法中对其进行配置，因为很多时候
    // 其父容器还不可用。比如WebApplicationContext。
    void setParent(@Nullable ApplicationContext parent);

    // 设置容器的Environment变量
    void setEnvironment(ConfigurableEnvironment environment);

    // 以COnfigurableEnvironment的形式返回此容器的环境，以使用户更好的进行配置
    @Override
    ConfigurableEnvironment getEnvironment();

    /**
     * 此方法一般在读取应用上下文配置的时候调用，用以向此容器中增加BeanFactoryPostProcessor。
     * 增加的Processor会在容器refresh的时候使用。
     */
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

    /**
     * 向容器增加一个ApplicationListener，增加的Listener用于发布上下文事件如refresh和shutdown等，
     * 需要注意的是，如果此上下文还没有启动，那么在此注册的Listener将会在上下文refresh的时候，全部被调用，
     * 如果上下文已经是active状态的了，就会在multicaster中使用
     */
    void addApplicationListener(ApplicationListener<?> listener);

    /**
     * 向容器中注入给定的Protocol resolver，允许多个实例同时存在。
     * 在此注册的每一个resolver都将会在上下的标准解析规则之前使用。因此，某种程度上来说
     * 这里注册的resolver可以覆盖上下文的resolver
     */
    void addProtocolResolver(ProtocolResolver resolver);

    /**
     * 加载或刷新资源配置文件（XML、properties, 与数据库相关的schema）。
     * 由于此方法是一个初始化方法，因此如果调用此方法失败的情况下，要将其已经创建的Bean销毁。
     * 换句话说，调用此方法以后，要么所有的Bean都实例化好了，要么就一个都没有实例化
     */
    void refresh() throws BeansException, IllegalStateException;

    /**
     * 向JVM注册一个回调函数，用以在JVM关闭时，销毁此应用上下文。
     */
    void registerShutdownHook();

    /**
     * 关闭此应用上下文，释放其所占有的所有资源和锁。并销毁其所有创建好的singleton Beans
     * 实现的时候，此方法不应该调用其父上下文的close方法，因为其父上下文具有自己独立的生命周期
     * 多次调用此方法，除了第一次，后面的调用应该被忽略。
     */
    @Override
    void close();

    /**
     * 检测此上下文是否时启动状态。
     * 也就是说在它关闭之前该上下文至少被refresh过一次
     *
     * 参见refresh()方法
     * 参见close()方法
     * 参见getBeanFactory()方法
     */
    boolean isActive();

    /**
     * 返回此应用上下文的底层存储bean的容器。
     * 千万不要使用此方法来对BeanFactory生成的Bean做后置处理，因为单例Bean在此之前已经生成，
     * 这种情况下应该使用BeanFactoryPostProcessor来在Bean生成之前对其进行处理。
     * 通常情况下，底层bean容器只有在上下文是激活的情况下才能使用。因此，在使用此方法前，可以调用
     * isActive来判断bean容器是否可用
     */
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
}
```



## 3.10、准备异常报告器

```java
exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
```

逻辑和之前实例化ApplicationContextInitializer和ApplicationListener的一样，一样调用的是 `getSpringFactoriesInstances` 方法来获取配置的异常类名称并实例化所有的异常处理类。

该异常报告处理类配置在 `spring-boot-2.2.1.RELEASE.jar!/META-INF/spring.factories` 这个配置文件里面。

```properties
# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers
```



## 3.11、准备ApplicationContext

```java
prepareContext(context, environment, listeners, applicationArguments, printedBanner);
```

来看下 `prepareContext()` 方法的源码：

```java
public class SpringApplication {

    // ...省略部分代码

    private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
                                SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, 
                                Banner printedBanner) {

        // 3.11.1、将environment保存到上一步创建的ConfigurableApplicationContext中
        context.setEnvironment(environment);
        // 3.11.2、ConfigurableApplicationContext创建后的后置处理逻辑
        postProcessApplicationContext(context);
        // 3.11.3、回调所有ApplicationContextInitializer的实现类的initialize(ConfigurableApplicationContext context)方法
        // org.springframework.boot.context.config.DelegatingApplicationContextInitializer  （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.ContextIdApplicationContextInitializer 				（spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer （spring-boot-2.2.1.RELEASE.jar）
        // org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
        applyInitializers(context);
        // 3.11.4、发布ApplicationContextInitializedEvent事件
        // org.springframework.boot.autoconfigure.BackgroundPreinitializer （spring-boot-autoconfigure-2.2.1.RELEASE.jar）
        // org.springframework.boot.context.config.DelegatingApplicationListener  （spring-boot-2.2.1.RELEASE.jar）
        listeners.contextPrepared(context);
        // 3.11.5、是否打印启动信息
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }
        // Add boot specific singleton beans
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }
        if (beanFactory instanceof DefaultListableBeanFactory) {
            // 注册bean时是否允许覆盖容器中已存在该名称的bean
            ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
        if (this.lazyInitialization) {
            context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
        }
        // Load the sources
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        load(context, sources.toArray(new Object[0]));
        listeners.contextLoaded(context);
    }

    // ...省略部分代码
}
```

下面具体分析每一步

### 3.11.1、将ConfigurationEnvironment存到ConfigurationApplicationContext中

```java
// 将environment保存到上一步创建的ConfigurableApplicationContext中
context.setEnvironment(environment);
```



### 3.11.2、ConfigurableApplicationContext创建后的后置处理逻辑

```java
// ConfigurableApplicationContext创建后的后置处理逻辑
 postProcessApplicationContext(context);
```

postProcessApplicationContext方法的源码如下：

```java
public class SpringApplication {

    // ...省略部分代码

    // 生成bean的名称时所使用的的生成器    
    private BeanNameGenerator beanNameGenerator;

    // 默认ApplicationConversionService应该添加到应用上下文(ApplicationContext)的Environment中
    private boolean addConversionService = true;

    // 构造方法及set方法均可赋值
    private ResourceLoader resourceLoader;

    // 设置在创建bean时bean的名称的生成器
    public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
        this.beanNameGenerator = beanNameGenerator;
    }

    // 设置ApplicationConversionService是否应该添加到应用上下文(ApplicationContext)的Environment中
    public void setAddConversionService(boolean addConversionService) {
        this.addConversionService = addConversionService;
    }

    // ConfigurableApplicationContext创建后的后置处理
    protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
        // 判断是否设置了bean名称的生成器
        if (this.beanNameGenerator != null) {
            context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                                                       this.beanNameGenerator);
        }
        // 是否指定了资源加载器，默认null
        if (this.resourceLoader != null) {
            if (context instanceof GenericApplicationContext) {
                ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
            }
            if (context instanceof DefaultResourceLoader) {
                ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
            }
        }
        // ApplicationConversionService是否应该添加到应用上下文(ApplicationContext)的Environment中，默认true
        if (this.addConversionService) {
            // 将ApplicationConversionService存到ConfigurableApplicationContext的底层存储bean的容器BeanFactory中
            context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
        }
    }
}
```

### 3.11.3、回调ApplicationContextInitializer.initialize(context)

```java
applyInitializers(context);
```
applyInitializers(context)的源码如下：

```java
public class SpringApplication {

    private List<ApplicationContextInitializer<?>> initializers;

    // 构造方法的源码细节前面分析过，参见第2节
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {

        // ...省略代码

        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

        // ...省略代码
    }

    // 回调所有ApplicationContextInitializer的实现类的initialize(ConfigurableApplicationContext context)方法
    protected void applyInitializers(ConfigurableApplicationContext context) {
        // 遍历spring.factories中key为org.springframework.context.ApplicationContextInitializer的实现类
        for (ApplicationContextInitializer initializer : getInitializers()) {
            // 针对给定的目标类解析给定泛型接口的单一参数，解析的时候会假设这个目标类实现了给定的泛型接口，
            // 并且可能为泛型接口的类型变量定义了具体类型。
            // resolveTypeArgument方法会返回解析到的泛型接口的单一参数类型，如果没解析到则返回nul
            Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                                                                            ApplicationContextInitializer.class);
            // 上一行和这一行的目的就是判断在spring.factories文件中key为org.springframework.context.ApplicationContextInitializer的值
            // 是否实现了ApplicationContextInitializer接口，并且指定了具体的泛型参数类型
            Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
            //回调ApplicationContextInitializer的initialize方法来初始化，方法传入参数为ConfigurableApplicationContext
            initializer.initialize(context);
        }
    }

    // 对initializers排序
    public Set<ApplicationContextInitializer<?>> getInitializers() {
        return asUnmodifiableOrderedSet(this.initializers);
    }

    // 对给定的集合elements使用AnnotationAwareOrderComparator排序
    private static <E> Set<E> asUnmodifiableOrderedSet(Collection<E> elements) {
        List<E> list = new ArrayList<>(elements);
        list.sort(AnnotationAwareOrderComparator.INSTANCE);
        return new LinkedHashSet<>(list);
    }
}
```

这里的getInitializers()方法返回的是当前类中的```initializer```属性，而```initializer```属性是在```SpringApplication```的构造函数中加载的所有```spring.factories```中key为```org.springframework.context.ApplicationContextInitializer```的实现类，具体的```ApplicationContextInitializer```的加载源码参见前文第2小节。

spring```org.springframework.context.ApplicationContextInitializer```的实现类出处如下：

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```properties
  # Application Context Initializers
  org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
  org.springframework.boot.context.ContextIdApplicationContextInitializer,\
  org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
  org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
  org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
  ```

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories``` 共2个

  ```properties
  # Initializers
  org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
  org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
  ```



### 3.11.4、发布ApplicationContextInitializedEvent事件

```java
listeners.contextPrepared(context);
```

`SpringApplicationRunListeners`类的contextPrepared方法源码如下：


```java
class SpringApplicationRunListeners {

    private final Log log;

    private final List<SpringApplicationRunListener> listeners;

    void contextPrepared(ConfigurableApplicationContext context) {
        for (SpringApplicationRunListener listener : this.listeners) {
            listener.contextPrepared(context);
        }
    }   
}
```

`SpringApplicationRunListener`的实现类只有`EventPublishingRunListener`，contextPrepared方法的源码如下：


```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;

    private final String[] args;

    private final SimpleApplicationEventMulticaster initialMulticaster;

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        this.initialMulticaster
            .multicastEvent(new ApplicationContextInitializedEvent(this.application, this.args, context));
    }
}
```



### 3.11.5、打印启动信息

```java
if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
}

protected void logStartupInfo(boolean isRoot) {
    if (isRoot) {
        new StartupInfoLogger(this.mainApplicationClass).logStarting(getApplicationLog());
    }
}

protected void logStartupProfileInfo(ConfigurableApplicationContext context) {
    Log log = getApplicationLog();
    if (log.isInfoEnabled()) {
        String[] activeProfiles = context.getEnvironment().getActiveProfiles();
        if (ObjectUtils.isEmpty(activeProfiles)) {
            String[] defaultProfiles = context.getEnvironment().getDefaultProfiles();
            log.info("No active profile set, falling back to default profiles: "
                     + StringUtils.arrayToCommaDelimitedString(defaultProfiles));
        }
        else {
            log.info("The following profiles are active: "
                     + StringUtils.arrayToCommaDelimitedString(activeProfiles));
        }
    }
}

protected Log getApplicationLog() {
    // 这个mainApplicationClass在当前类的构造函数中被赋值了
    // 赋值代码如下：this.mainApplicationClass = deduceMainApplicationClass();
    if (this.mainApplicationClass == null) {
        return logger;
    }
    // 返回apache的commons-logging的Log对象
    return LogFactory.getLog(this.mainApplicationClass);
}
```




## 3.12、刷新ApplicationContext

```java
refreshContext(context);
```

这个主要是刷新 Spring 的应用上下文，源码如下:

```java
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}
```



## 3.13、ApplicationContext刷新后置处理

```java
afterRefresh(context, applicationArguments);
```

这个方法的源码是空的，目前可以做一些自定义的后置处理操作。

```
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
}
```

## 3.14、停止StopWatch

```
stopWatch.stop();
```

```java
public void stop() throws IllegalStateException {
    if (this.currentTaskName == null) {
        throw new IllegalStateException("Can't stop StopWatch: it's not running");
    }
    long lastTime = System.nanoTime() - this.startTimeNanos;
    this.totalTimeNanos += lastTime;
    this.lastTaskInfo = new TaskInfo(this.currentTaskName, lastTime);
    if (this.keepTaskList) {
        this.taskList.add(this.lastTaskInfo);
    }
    ++this.taskCount;
    this.currentTaskName = null;
}
```

计时监听器停止，并统计一些任务执行信息。



## 3.15、输出日志记录执行主类名、时间信息

```java
if (this.logStartupInfo) {
    new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
}
```



## 3.16、发布ApplicationStartedEvent事件

```java
listeners.started(context);
```

触发所有 SpringApplicationRunListener 监听器的 started 事件方法。

```
class SpringApplicationRunListeners {

	private final Log log;

	private final List<SpringApplicationRunListener> listeners;
	
	...省略
	
    void started(ConfigurableApplicationContext context) {
       for (SpringApplicationRunListener listener : this.listeners) {
          listener.started(context);
       }
    }
}
```

`SpringApplicationRunListener`的实现类只有`EventPublishingRunListener`，started源码如下：



```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;

	private final String[] args;

	private final SimpleApplicationEventMulticaster initialMulticaster;
    
    @Override
	public void started(ConfigurableApplicationContext context) {
		context.publishEvent(new ApplicationStartedEvent(this.application, this.args, context));
	}
}
```

## 3.17、执行所有 Runner 运行器

```java
callRunners(context, applicationArguments);
```

callRunners方法源码如下：

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}

private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
    try {
        (runner).run(args);
    }
    catch (Exception ex) {
        throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
    }
}

private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
    try {
        (runner).run(args.getSourceArgs());
    }
    catch (Exception ex) {
        throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
    }
}
```

执行所有 `ApplicationRunner` 和 `CommandLineRunner` 这两种运行器。



## 3.18、发布ApplicationReadyEvent事件

```
try {
   listeners.running(context);
}
catch (Throwable ex) {
   handleRunFailure(context, ex, exceptionReporters, null);
   throw new IllegalStateException(ex);
}
```

触发所有 SpringApplicationRunListener 监听器的 running 事件方法。

```java
class SpringApplicationRunListeners {

	private final Log log;

	private final List<SpringApplicationRunListener> listeners;
	
	...省略
	
   	void running(ConfigurableApplicationContext context) {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.running(context);
		}
	}
}
```

`SpringApplicationRunListener`的实现类只有`EventPublishingRunListener`，running源码如下：

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;

    private final String[] args;

    private final SimpleApplicationEventMulticaster initialMulticaster;

    @Override
    public void running(ConfigurableApplicationContext context) {
        context.publishEvent(new ApplicationReadyEvent(this.application, this.args, context));
    }
}
```



## 3.19、返回应用上下文

```
return context;
```