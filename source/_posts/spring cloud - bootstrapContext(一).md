---
title: spring cloud - bootstrapContext(一)
date: 2020-01-06 10:11:23
tags: spring cloud
categories: spring cloud
---

# 1、前言

springcloud是基于springboot开发的，所以读者在阅读此文前最好已经了解了[springboot的工作原理](https://www.cnblogs.com/question-sky/p/9360722.html)。本文将不阐述springboot的工作逻辑。

在整个 Spring Boot 启动的生命周期过程中，有一个阶段是 prepare environment。在这个阶段，会publish 一个 `ApplicationEnvironmentPreparedEvent`，通知所有对这个事件感兴趣的 Listener,提供对 Environment 做更多的定制化的操作。Spring Cloud 定义了一个`BootstrapApplicationListener`，在 `BootstrapApplicationListener` 的处理过程中会创建spring cloud的ApplicationContext。

# 2、spring cloud context

springboot cloud context在官方的文档中在第一点被提及，是用户ApplicationContext的父级上下文，笔者称呼为**BootstrapContext**。根据springboot的加载机制，很多第三方以及重要的Configuration配置均是保存在了**spring.factories**文件中。

笔者翻阅了**spring-cloud-context**模块下的对应文件，见如下:

```properties
# AutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
org.springframework.cloud.autoconfigure.LifecycleMvcEndpointAutoConfiguration,\
org.springframework.cloud.autoconfigure.RefreshAutoConfiguration,\
org.springframework.cloud.autoconfigure.RefreshEndpointAutoConfiguration,\
org.springframework.cloud.autoconfigure.WritableEnvironmentEndpointAutoConfiguration
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
org.springframework.cloud.bootstrap.LoggingSystemShutdownListener,\
org.springframework.cloud.context.restart.RestartListener
# Bootstrap components
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration,\
org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration,\
org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration

```

涉及的主要分三类，笔者优先分析**监听器**，其一般拥有更高的优先级并跟其他两块有一定的关联性。
除了日志监听器笔者不太关注，其余两个分步骤来分析

## 2.1、RestartListener

重启监听器，应该是用于刷新上下文的，直接查看下其复写的方法

```java
public class RestartListener implements SmartApplicationListener {

	private ConfigurableApplicationContext context;

	private ApplicationPreparedEvent event;
    
    @Override
	public void onApplicationEvent(ApplicationEvent input) {
         // 判断是否是ApplicationPreparedEvent事件, 如果是先缓存context
		if (input instanceof ApplicationPreparedEvent) {
			this.event = (ApplicationPreparedEvent) input;
			if (this.context == null) {
				this.context = this.event.getApplicationContext();
			}
		}
         // 上下文刷新结束事件(ContextRefreshedEvent)，重新发布ApplicationPreparedEvent事件
		else if (input instanceof ContextRefreshedEvent) {
			if (this.context != null && input.getSource().equals(this.context)
					&& this.event != null) {
                  // 注意这里的this.event的赋值是在前面的ApplicationPreparedEvent事件发生时赋值的
				this.context.publishEvent(this.event);
			}
		}
		else {
            // 上下文关闭事件传播至此，则开始清空所拥有的对象
			if (this.context != null && input.getSource().equals(this.context)) {
				this.context = null;
				this.event = null;
			}
		}
	}
}
```

上述的刷新事件经过查阅，与**org.springframework.cloud.context.restart.RestartEndpoint**类有关，这个就后文再分析好了



## 2.2、BootstrapApplicationListener

按照顺序分析此监听器

**1、优先看下其类结构**

```java
public class BootstrapApplicationListener
    implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered {
}
```

此监听器是用于响应**ApplicationEnvironmentPreparedEvent**应用环境变量预初始化事件，**表明BootstrapContext的加载时机在用户上下文(spring boot的ApplicationContext)之前**，且其**加载顺序比ConfigFileApplicationListener监听器超前**，这点稍微强调下，监听器的具体执行顺序见我的另一篇文章[Spring Boot启动流程分析](https://calebzhao.github.io/2019/12/30/Spring%20Boot启动流程分析/#3-6-4、发布ApplicationEnvironmentPreparedEvent事件)。

**2、接下来分析下其复写的方法*onApplicationEvent(ApplicationEnvironmentPreparedEvent event)**

```java
@Override
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    // 获取spring boot的环境变量对象
    ConfigurableEnvironment environment = event.getEnvironment();
    // 读取spring.cloud.bootstrap.enabled环境属性，默认为true。可通过系统变量设置
    // 这行代码的意图是用于配置是否启用spring cloud
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
        // 说明当前是spring cloud启动过程中发布的ApplicationEnvironmentPreparedEvent事件，不能再次启动spring cloud了
        return;
    }
    ConfigurableApplicationContext context = null;
    // 从命令行参数、系统属性、系统环境变量解析spring.cloud.bootstrap.name的值，如果都未指定则使用默认名称bootstrap
    String configName = environment.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");

    // 寻找当前环境是否已存在BootstrapContext，
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
    for (ApplicationContextInitializer<?> initializer : event.getSpringApplication()
         .getInitializers()) {
        if (initializer instanceof ParentContextApplicationContextInitializer) {
            context = findBootstrapContext(
                (ParentContextApplicationContextInitializer) initializer,
                configName);
        }
    }
    // 如果spring cloud的BootstrapContext还没有被创建，则开始创建
    if (context == null) {
        context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);
        
        // 注册注销监听器
        event.getSpringApplication()
            .addListeners(new CloseContextOnFailureApplicationListener(context));
    }

      // 将spring cloud的applicationContext中的ApplicationContextInitializers添加到spring boot的Context上
    apply(context, event.getSpringApplication(), environment);
}

// 注意调用apply时spring cloud已经启动完成了，而当前的spring boot还处于ApplicationEnvironmentPreparedEvent事件执行阶段
private void apply(ConfigurableApplicationContext context, SpringApplication application, ConfigurableEnvironment environment) {
    @SuppressWarnings("rawtypes")
    // 从spring cloud的applicationContext中获取bean类型为ApplicationContextInitializer的事件监听器
    // spring cloud启动过程中会注册所有的@Bean注解声明的组件, 
    List<ApplicationContextInitializer> initializers = getOrderedBeansOfType(context, ApplicationContextInitializer.class);
    // 将spring cloud的applicationContext中的ApplicationContextInitializers添加到spring boot的Context上
    application.addInitializers(initializers
                                .toArray(new ApplicationContextInitializer[initializers.size()]));
    addBootstrapDecryptInitializer(application);
}
```

逻辑很简单，上面的代码注释已经给出每一步详细说明，这里再梳理下：

- **spring.cloud.bootstrap.enabled** 用于配置是否启用BootstrapContext，默认为true。可通过命令行参数、系统属性、系统环境变量设定
- **spring.cloud.bootstrap.name** 用于加载bootstrap对应配置文件的别名，默认为bootstrap，在ConfigFileApplicationListener事件中会使用该名称加载配置文件
- BootstrapContext上的beanType为**ApplicationContextInitializer**类型的bean对象集合会被注册至用户的Context上



**3、重点看下BootstrapContext的创建过程，源码比较长**

```java
private ConfigurableApplicationContext bootstrapServiceContext(
    ConfigurableEnvironment environment, final SpringApplication application, String configName) {
    // 创建spring cloud 的一个空的environment
    StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
    // 获取上面创建spring cloud 的可变属性源
    MutablePropertySources bootstrapProperties = bootstrapEnvironment.getPropertySources();
    // 移除spring cloud的所有属性源，让spring cloud的environment变成一个不包含任何属性源的空environment
    for (PropertySource<?> source : bootstrapProperties) {
        bootstrapProperties.remove(source.getName());
    }

    // 从spring boot的environment获取"spring.cloud.bootstrap.location"属性, 一般通过命令行参数、jvm系统属性、系统环境变量设置，默认为空
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
        builder.main(application.getMainApplicationClass());
    }
    if (environment.getPropertySources().contains("refreshArgs")) {
        // If we are doing a context refresh, really we only want to refresh the
        // Environment, and there are some toxic listeners (like the
        // LoggingApplicationListener) that affect global static state, so we need a
        // way to switch those off.
        builderApplication.setListeners(filterListeners(builderApplication.getListeners()));
    }
    // 增加入口类BootstrapImportSelectorConfiguration
    builder.sources(BootstrapImportSelectorConfiguration.class);
    // 关键代码，创建spring cloud自身的ApplicationContext，又会把spring boot的SpringApplication.run()方法的流程全部走一遍
    final ConfigurableApplicationContext context = builder.run();

    // 设置spring cloud的ApplicationContext的id为"bootstrap"
    context.setId("bootstrap");
    // 这里的addAncestorInitializer方法的第1个参数application是spring boot的SpringApplication对象
    // 而第2个参数context是上一行代码返回的spring cloud的ApplicationContext
    // 这里的意图是配置spring cloud的bootstrapContext为spring boot的Context的父上下文
    addAncestorInitializer(application, context);
    // spring cloud的环境中现在有一些属性是我们不想在父ApplicationContext中看到的，所以把它移除(稍后会添加回来)
    bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
    // 合并spring cloud中name为"defaultProperties"属性源至spring boot中，注意同名属性不覆盖
    mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
    // 返回spring cloud的ApplicationContext
    return context;
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

此处也对上述的代码作下简单的小结：

- **spring.cloud.bootstrap.location**变量用于配置bootstrapContext配置文件的加载路径，可用System设置，默认则采取默认的文件搜寻路径；与*spring.cloud.bootstrap.name*搭配使用
- bootstrapContext对应的activeProfiles可采用*spring.active.profiles*系统变量设置，注意是System变量。当然也可以通过**bootstrap.properties/bootstrap.yml**配置文件设置
- bootstrapContext的重要入口类为**BootstrapImportSelectorConfiguration**，此也是下文的分析重点
- bootstrapContext的contextId为*bootstrap*。即使配置了*spring.application.name*属性也会被设置为前者，且其会被设置为**用户Context的父上下文**
- bootstrap.(yml | properties)上的配置会被合并至用户级别的Environment中的**defaultProperties**集合中，且其相同的KEY会被丢弃，不同KEY会被保留。即其有最低的属性优先级

通过上述的代码均可以得知，bootstrapContext也是通过springboot常见的`SpringApplication`方式来创建的，但其肯定有特别的地方。
特别之处就在**BootstrapImportSelectorConfiguration**类，其也与上述`spring.factories`文件中**org.springframework.cloud.bootstrap.BootstrapConfiguration**的Key有直接的关系，我们下文重点分析。

## 2.3、BootstrapImportSelector

首先回顾bootstrapServiceContext()方法中的BootstrapImportSelectorConfiguration设置过程。

```java
private ConfigurableApplicationContext bootstrapServiceContext(){
    // ...省略代码
    
    // 增加入口类BootstrapImportSelectorConfiguration
    builder.sources(BootstrapImportSelectorConfiguration.class);
    
    // ...省略代码
}
    
```

承接前文监听器对bootstrapContext创建的引导，可以看到到其主要入口类为`BootstrapImportSelectorConfiguration`。下文将基于此类进行简单的分析。

其源码如下：

```java
@Configuration(proxyBeanMethods = false)
@Import(BootstrapImportSelector.class)
public class BootstrapImportSelectorConfiguration {

}
```

嗯，引入了延迟加载类**BootstrapImportSelector**，那就继续往下看下此会延迟加载哪些类，直接去观察其主方法。

```java
public class BootstrapImportSelector implements EnvironmentAware, DeferredImportSelector {
    
    private Environment environment;

    private MetadataReaderFactory metadataReaderFactory = new CachingMetadataReaderFactory();

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        // 加载classpath路径下所有spring.factories文件中以org.springframework.cloud.bootstrap.BootstrapConfiguration作为Key的所有类
        List<String> names = new ArrayList<>(SpringFactoriesLoader.loadFactoryNames(BootstrapConfiguration.class, classLoader));
        // 支持通过设置spring.cloud.bootstrap.sources属性来加载指定的类
        names.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(
            this.environment.getProperty("spring.cloud.bootstrap.sources", ""))));

        List<OrderedAnnotatedElement> elements = new ArrayList<>();
        // 创建用于排序的OrderedAnnotatedElement
        for (String name : names) {
            try {
                elements.add(
                    new OrderedAnnotatedElement(this.metadataReaderFactory, name));
            }
            catch (IOException e) {
                continue;
            }
        }
        // 对以org.springframework.cloud.bootstrap.BootstrapConfiguration作为Key的所有类进行就地排序
        AnnotationAwareOrderComparator.sort(elements);

        // 得到排序后的名称
        String[] classNames = elements.stream().map(e -> e.name).toArray(String[]::new);

        return classNames;
    }
    
    // ...省略后续代码

}
```

上述的代码很简单，其会去加载classpath路径下所有**spring.factories**文件中以**org.springframework.cloud.bootstrap.BootstrapConfiguration**作为Key的所有类；
同时springcloud也支持通过设置**spring.cloud.bootstrap.sources**属性来加载指定类。

下面就先以springcloud context板块内的**spring.factories**作为分析的源头

- ```spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories``` 共4个

    ```properties

    # Bootstrap components
    org.springframework.cloud.bootstrap.BootstrapConfiguration=\
    org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration,\
    org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration,\
    org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
    org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
    ```



- ```spring-cloud-netflix-eureka-client-2.2.0.RELEASE.jar!\META-INF\spring.factories```共1个

  ```properties
  org.springframework.cloud.bootstrap.BootstrapConfiguration=\
  org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
  ```

在分析上述的源码之前，笔者必须得清楚**现在bootstrapContext加载的配置文件默认为bootstrap.properties抑或是bootstrap.yml，其属性已经被加入至相应的Environment对象中**。基于此我们再往下走，以免犯糊涂。

### 2.3.1、PropertySourceBootstrapConfiguration

**配置源的加载**，此Configuration主要用于确定是否外部加载的配置属性复写Spring内含的环境变量。注意其是**ApplicationContextInitializer**接口的实现类，**前文已经提到，bootstrapContext上的此接口的bean类都会被注册至子级的SpringApplication对象上**。
直接看下主要的代码片段吧

```java
/**
* spring cloud通过 PropertySourceLocator 加载外部环境属性源
*
*/
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
    ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

    // BOOTSTRAP_PROPERTY_SOURCE_NAME常量的值为"bootstrapProperties"
    public static final String BOOTSTRAP_PROPERTY_SOURCE_NAME = BootstrapApplicationListener.BOOTSTRAP_PROPERTY_SOURCE_NAME + "Properties";

    private int order = Ordered.HIGHEST_PRECEDENCE + 10;

    // 注意这里注入了PropertySourceLocator
    @Autowired(required = false)
    private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

    public void setPropertySourceLocators(Collection<PropertySourceLocator> propertySourceLocators) {
        this.propertySourceLocators = new ArrayList<>(propertySourceLocators);
    }

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 创建1个具有复合功能的属性源
        CompositePropertySource composite = new OriginTrackedCompositePropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME);
        // 对propertySourceLocators排序
        AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
        // 一个flag, 标识是否加载到外部属性源
        boolean empty = true;
        // 此处为子级的环境变量对象
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        
         // 通过PropertySourceLocator接口去加载外部配置
        for (PropertySourceLocator locator : this.propertySourceLocators) {
            PropertySource<?> source = null;
            // 重点：加载外部配置， spring cloud config、nacos的集合就是通过PropertySourceLocator来实现的
            source = locator.locate(environment);
            if (source == null) {
                continue;
            }
            logger.info("Located property source: " + source);
            // 把从外部环境加载到的属性源加到前面定义的复合属性源中
            composite.addPropertySource(source);
            empty = false;
        }
        // 判断是否从外部环境加载到任何1个属性源
        if (!empty) {
            // 获取spring boot的可变属性源
            MutablePropertySources propertySources = environment.getPropertySources();
            String logConfig = environment.resolvePlaceholders("${logging.config:}");
            LogFile logFile = LogFile.get(environment);
            // 判断spring boot的environment中是否存在name为"bootstrapProperties"的属性源
            if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                // 移除spring boot的environment中name为"bootstrapProperties"的属性源
                propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
            }
            // 确定属性读取的先后顺序
            insertPropertySources(propertySources, composite);
            // 日志相关，不管
            reinitializeLoggingSystem(environment, logConfig, logFile);
            // 设置日志级别，不管
            setLogLevels(applicationContext, environment);
            handleIncludedProfiles(environment);
        }
    }
    
    /**
    * 将远程属性源以适合的顺序插入到本地environment中：
    * 插入前要确定远程属性源与本地属性源的优先级哪个高(属性读取的先后顺序)，也就是通过key获取value时先读取谁的配置。
    *
    * 这里的相关代码与spring.cloud.config.allowOverride、spring.cloud.config.overrideNone、spring.cloud.config.overrideSystemProperties配置相关
    *
    * @param propertySources 它是spring boot的可变属性源
    * @parm composite 复合属性源，里面的属性源是spring cloud从外部环境加载到的
    */
    private void insertPropertySources(MutablePropertySources propertySources,
			CompositePropertySource composite) {
         // 将从外部环境加载的属性源添加到新创建的可变属性源中
		MutablePropertySources incoming = new MutablePropertySources();
		incoming.addFirst(composite);
        
         // 从外部环境加载的属性源中获取key为"spring.cloud.config"的值，并将其值封装到PropertySourceBootstrapProperties类中
		PropertySourceBootstrapProperties remoteProperties = new PropertySourceBootstrapProperties();
		Binder.get(environment(incoming)).bind("spring.cloud.config", Bindable.ofInstance(remoteProperties));
         // 关键代码了， 如果配置影响了远程配置与本地配置的优先级：
         // spring.cloud.config.allowOverride=true
         // spring.cloud.config.overrideNone=true
         // spring.cloud.config.overrideSystemProperties=false
        
         // remoteProperties.isAllowOverride() 
         // 1、远程配置是否允许被本地配置覆盖（是否允许允许本地属性配置覆盖远程属性配置）
        
         //  remoteProperties.isOverrideNone()
         // 2、远程配置允许被本地配置覆盖的情况下，判断远程属性的任意配置是否都允许被本地配置覆盖
        
         // remoteProperties.isOverrideSystemProperties()
         // 3、远程配置允许被本地配置覆盖的情况下，远程属性不能被本地配置任意覆盖， 再判断远程配置是否可以覆盖本地系统环境变量
		if (! remoteProperties.isAllowOverride() 
            	||  (!remoteProperties.isOverrideNone() && remoteProperties.isOverrideSystemProperties())) {
            
			propertySources.addFirst(composite);
			return;
		}
         // 判断远程属性的任意配置是否都允许被本地配置覆盖
		if (remoteProperties.isOverrideNone()) {
             // 运行到这里说明：远程属性的任意配置都允许被本地配置覆盖， 也就是说远程配置的优先级低于本地配置，
             // 那么把远程配置添加到environment的最后(属性源的位置越靠前代表其优先级越高，越靠后则优先级越低)
			propertySources.addLast(composite);
			return;
		}
        
         // 本地环境是否包含系统环境变量属性源， SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME常量的值为"systemEnvironment"
		if (propertySources
				.contains(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME)) {
             // 远程配置是否可以覆盖本地系统环境变量配置
			if (!remoteProperties.isOverrideSystemProperties()) {
                  // 远程配置不能覆盖本地系统环境变量配置，也就是说本地系统环境变量的优先级高于远程配置，
                  // 那么把远程属性源添加在本地系统环境变量属性源的后面
				propertySources.addAfter(
						StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
						composite);
			}
			else {
                 // 运行到这里表示远程配置可以覆盖本地系统环境变量，也就是说远程配置的优先级高于本地系统环境变量，
                 // 那么把远程属性源添加到本地系统环境变量属性源的前面
				propertySources.addBefore(
						StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
						composite);
			}
		}
		else {
			propertySources.addLast(composite);
		}
	}

}
```

上述的代码就涉及两点：

- 一个是通过*PropertySourceLocator*接口加载外部配置
- 一个是用于解析以**spring.cloud.config**为开头的**PropertySourceBootstrapProperties**属性，默认情况下，外部配置比内部变量有更高的优先级。



### 2.3.2、ConfigurationPropertiesRebinderAutoConfiguration

通过命名便会发现其跟刷新属性的功能有关，先看下其类结构：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(ConfigurationPropertiesBindingPostProcessor.class)
public class ConfigurationPropertiesRebinderAutoConfiguration
		implements ApplicationContextAware, SmartInitializingSingleton {
    
    
}
```

 上述代码表示其依据于当前类环境存在*ConfigurationPropertiesBindingPostProcessor*Bean这个bean才会被应用，仔细查阅了下，发现只要有使用到**@EnableConfigurationProperties**注解即就会被注册。看来此配置跟*ConfigurationProperties*注解也有一定的关联性。

下面看看ConfigurationPropertiesBindingPostProcessor的源码：

- ConfigurationPropertiesBindingPostProcessor

  ```java
  public class ConfigurationPropertiesBindingPostProcessor
  		implements BeanPostProcessor, PriorityOrdered, ApplicationContextAware, InitializingBean {
      
      // ...省略代码
      
      // 实现了InitializingBean接口， 在bean的属性初始化后会调用该方法
      @Override
  	public void afterPropertiesSet() throws Exception {
  		this.registry = (BeanDefinitionRegistry) this.applicationContext.getAutowireCapableBeanFactory();
  		this.binder = ConfigurationPropertiesBinder.get(this.applicationContext);
  	}
      
      // 实现了BeanPostProcessor
      @Override
  	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  		bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
  		return bean;
  	}
  
  	private void bind(ConfigurationPropertiesBean bean) {
  		if (bean == null || hasBoundValueObject(bean.getName())) {
  			return;
  		}
  		Assert.state(bean.getBindMethod() == BindMethod.JAVA_BEAN, "Cannot bind @ConfigurationProperties for bean '"
  				+ bean.getName() + "'. Ensure that @ConstructorBinding has not been applied to regular bean");
  		try {
  			this.binder.bind(bean);
  		}
  		catch (Exception ex) {
  			throw new ConfigurationPropertiesBindException(bean, ex);
  		}
  	}
  }
  ```

  本文就罗列笔者比较关注的几个地方

  1.ConfigurationPropertiesRebinder对象的创建

  ```java
  @Bean
  @ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
  public ConfigurationPropertiesRebinder configurationPropertiesRebinder(
      ConfigurationPropertiesBeans beans) {
      ConfigurationPropertiesRebinder rebinder = new ConfigurationPropertiesRebinder(
          beans);
      return rebinder;
  }
  ```

  此类读者可以自行翻阅代码，可以发现其暴露了JMX接口以及监听了springcloud context自定义的*EnvironmentChangeEvent*事件。看来其主要用来刷新ApplicationContext上的beans(**含@ConfigurationProperties注解**)对象集合

  2.ConfigurationPropertiesBeans对象的创建

  BeanPostProcessor将PropertySources绑定到使用@ConfigurationProperties注释的bean。

  ```java
  @Bean
  @ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
  public ConfigurationPropertiesBeans configurationPropertiesBeans() {
      return new ConfigurationPropertiesBeans();
  }
  
  ```

  配合@ConfigurationProperties注解，其会缓存ApplicationContext上的所有含有*ConfigurationProperties*注解的bean。与第一点所提的**ConfigurationPropertiesRebinder**对象搭配使用

  

  3.实例化结束后刷新父级ApplicationContext上的属性

  ```java
  @Override
  public void afterSingletonsInstantiated() {
      // After all beans are initialized explicitly rebind beans from the parent
      // so that changes during the initialization of the current context are
      // reflected. In particular this can be important when low level services like
      // decryption are bootstrapped in the parent, but need to change their
      // configuration before the child context is processed.
      if (this.context.getParent() != null) {
          // TODO: make this optional? (E.g. when creating child contexts that prefer to
          // be isolated.)
          ConfigurationPropertiesRebinder rebinder = this.context
              .getBean(ConfigurationPropertiesRebinder.class);
          for (String name : this.context.getParent().getBeanDefinitionNames()) {
              rebinder.rebind(name);
          }
      }
  }
  ```

### 2.3.3、EncryptionBootstrapConfiguration

与属性读取的加解密有关，跟JDK的keystore也有一定的关联，具体就不去解析了。读者可自行分析



# 3、Bean加载问题

此处本文插入这个Bean的加载问题，因为笔者发现SpringApplication会被调用两次，那么ApplicationContext实例也会被创建两次。那么基于*@Configuration*修饰过的自定义的Bean是不是也会被加载两次呢？？

经过在cloud环境下编写了一个简单的Bean

```java
package com.example.clouddemo;

import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Configuration;

/**
 * @author nanco
 * -------------
 * -------------
 * @create 19/8/20
 */
@Configuration
public class TestApplication implements ApplicationContextAware {

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("application parent context id: " + applicationContext.getParent().getId());
        System.out.println("application context id: " + applicationContext.getId());
    }
}
```

且在application.properties文件中指定*spring.application.name=springChild*以及bootstrap.properties文件中也指定*spring.application.name=springBootstrap*

运行main方法之后打印的关键信息如下:

```
application parent context id: bootstrap
application context id: springBootstrap-1
```

经过在*org.springframework.boot.context.ContextIdApplicationContextInitializer*类上进行断点调试发现**只有用户级ApplicationContext被创建的过程中会实例化用户自定义Bean**。也就是说**bootstrapContext并不会去实例化用户自定义的Bean**，这样就很安全。

**那么为何如此呢???**

其实很简单，**因为bootstrapContext指定的source类只有BootstrapImportSelectorConfiguration，并没有用户编写的启动类，也就无法影响用户级别Context的Bean加载实例化了，并且该类上无@EnableAutoConfiguration、@ComponentScan注解，表明其也不会去扫描我们自己的包以及处理spring.factories文件中@EnableAutoConfiguration注解key对应的配置集合**， 而用户自己编写的启动类上都会有@SpringBootApplication注解，该注解是一个复合注解，里面会引入了@EnableAutoConfiguration、@ComponentScan、@SpringBootConfiguration，它会扫描我们自己的包以及处理spring.factories文件中@EnableAutoConfiguration注解key对应的配置集合。