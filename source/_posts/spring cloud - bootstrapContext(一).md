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
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
    ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

    // BOOTSTRAP_PROPERTY_SOURCE_NAME常量的值为"bootstrapProperties"
    public static final String BOOTSTRAP_PROPERTY_SOURCE_NAME = BootstrapApplicationListener.BOOTSTRAP_PROPERTY_SOURCE_NAME + "Properties";

    private int order = Ordered.HIGHEST_PRECEDENCE + 10;

    @Autowired(required = false)
    private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

    public void setPropertySourceLocators(Collection<PropertySourceLocator> propertySourceLocators) {
        this.propertySourceLocators = new ArrayList<>(propertySourceLocators);
    }

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        CompositePropertySource composite = new OriginTrackedCompositePropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME);
        AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
        boolean empty = true;
        // 此处为子级的环境变量对象
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        
         // 通过PropertySourceLocator接口去加载外部配置
        for (PropertySourceLocator locator : this.propertySourceLocators) {
            PropertySource<?> source = null;
            // 加载外部配置
            source = locator.locate(environment);
            if (source == null) {
                continue;
            }
            logger.info("Located property source: " + source);
            composite.addPropertySource(source);
            empty = false;
        }
        if (!empty) {
            MutablePropertySources propertySources = environment.getPropertySources();
            String logConfig = environment.resolvePlaceholders("${logging.config:}");
            LogFile logFile = LogFile.get(environment);
            if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
            }
            // 确定属性读取的先后顺序
            insertPropertySources(propertySources, composite);
            reinitializeLoggingSystem(environment, logConfig, logFile);
            setLogLevels(applicationContext, environment);
            handleIncludedProfiles(environment);
        }
    }

}
```



