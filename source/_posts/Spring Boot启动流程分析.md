---
title: Spring Boot启动流程分析
date: 2019-12-30 08:22:37
tags: spring boot
categories: spring boot
---

# 1、前言

学习过springboot的都知道，在Springboot的main入口函数中调用SpringApplication.run(DemoApplication.class,args)函数便可以启用SpringBoot应用程序，跟踪一下SpringApplication源码可以发现，最终还是调用了SpringApplication的动态run函数。

下面以SpringBoot2.2.1.RELEASE为例简单分析一下运行过程。



我们在编写一个spring boot应用时通常启动的方式是通过```SpringApplication.run(xxx.class, args)```来启动的，

```java
@SpringBootApplication
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```



# 2、分析 SpringApplication构造函数

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

可知这个构造器类的初始化包括以下 7 个过程:

## 2.1、资源初始化资源加载器为 null
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

```ApplicationContextInitializer``` 的作用是什么？源码如下。

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

```java
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
    listeners.starting();
    
    try {
        // 5、实例化ApplicationArguments对象，初始化默认应用参数类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
       
        // 6、 创建Environment （web环境 or 标准环境）+配置Environment，主要是把run方法的参数配置
        // 到Environment  发布【ApplicationEnvironmentPreparedEvent】事件
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        
        // 7、打印banner，SpringBoot启动时，控制台输出的一个歪歪扭扭的很不清楚的Spring几个大字母，
        // 也可以自定义，参考博客：http://majunwei.com/view/201708171646079868.html
        Banner printedBanner = printBanner(environment);
        
        // 8、 根据不同environment创建应用上下文（返回ConfigurableApplicationContext的子类实例）
        context = createApplicationContext();
        
        // 9、异常处理，准备异常报告器
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                      new Class[] { ConfigurableApplicationContext.class }, context);
      
        // 10、 上下文相关预处理，发布【ApplicationPreparedEvent】事件 
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
        
        // 15、执行所有runlisteners的started方法，发布【ApplicationStartedEvent】事件
        listeners.started(context);
        
        // 16: 遍历执行CommandLineRunner和ApplicationRunner
     　 // 如果需要在SpringBoot应用启动后运行一些特殊的逻辑，可以通过实现ApplicationRunner
        // 或CommandLineRunner接口中的run方法，该自定义类的run方法会在此处统一调用
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 17、遍历前面14行获得SpringApplicationRunListeners局部变量中所维护的listeners属性,
        // 执行所有SpringApplicationRunListener的running方法，发布【ApplicationReadyEvent】事件
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

所以，我们可以按以下几步来分解 run 方法的启动过程。

## 3.1、创建并启动计时监控类StopWatch

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

## 3.2、初始化应用上下文和异常报告集合SpringBootExceptionReporter

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

## 3.4、创建SpringApplicationRunListeners并发布ApplicationStartingEvent应用启动事件

### 3.4.1、getRunListeners()方法
```java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting();
```

来看下创建 SpringApplicationRunListeners运行监听器相关的源码：

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger,
                                             getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

可以看到返回了SpringApplicationRunListeners对象

### 3.4.1、创建SpringApplicationRunListeners

返回的SpringApplicationRunListeners这个类的源码如下：

```java
class SpringApplicationRunListeners {
    private final Log log;

	private final List<SpringApplicationRunListener> listeners;

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
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    
    // SpringApplication.run(xxx.class, args)运行时内部创建的SpringApplication实例，
    // 构造方法实例化时设置了List<ApplicationContextInitializer<?>> initializers及List<ApplicationListener<?>> listeners属性
	private final SpringApplication application;

    // SpringApplication.run(xxx.class, args)传入的args参数
	private final String[] args;

	private final SimpleApplicationEventMulticaster initialMulticaster;
    
    public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
    
    @Override
	public void starting() {
		this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
	}
}
```

### 3.4.5、SimpleApplicationEventMulticaster.multicastEvent(ApplicationEvent event)

SimpleApplicationEventMulticaster类的源码如下：

```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
	@Nullable
	private Executor taskExecutor;

	@Nullable
	private ErrorHandler errorHandler;
    
    public SimpleApplicationEventMulticaster() {
        
	}
    
    @Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}
    
    @Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
    
    protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}
    
    private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
             // onApplicationEvent方法就是我们平时使用ApplicationLisenter接口时要实现的方法
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



## 3.6、创建Environment，并发布ApplicationEnvironmentPreparedEvent事件

```
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
configureIgnoreBeanInfo(environment);
```

下面我们主要来看下准备环境的 `prepareEnvironment` 源码：

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                   ApplicationArguments applicationArguments) {
     // 3.6.1、 获取（或者创建）应用环境
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    
     // 3.6.2、 配置应用环境
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                                                                                               deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

### 3.6.1、获取（或者创建）应用环境

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

### 3.6.2、 配置应用环境
```java
 configureEnvironment(environment, applicationArguments.getSourceArgs());
```

configureEnvironment()实现如下：
```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    // SpringApplication的成员变量，默认值为true。
    if (this.addConversionService) {
        // 创建ApplicationConversionService并注册默认的Converter、Formatter类型转换器实现，spring mvc中经常使用ConversionService
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        // 将ConversionService存到ConfigurableEnvironment中
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

#### 3.6.2.1、创建ConversionService并存到Environment中

- ApplicationConversionService.getSharedInstance()源码如下：

```java
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
}
```



#### 3.6.2.2、初始化PropertySource

