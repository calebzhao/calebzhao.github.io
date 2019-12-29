---
title: spring-cloud-openfeign源码分析
date: 2019-12-29 12:32:44
tags: 
- feign
- spring cloud
categories: spring cloud
---


# 1、简介
feign是一个声明式的HTTP客户端，spring-cloud-openfeign将feign集成到spring boot中，在接口上通过注解声明Rest协议，将http调用转换为接口方法的调用，使得客户端调用http服务更加简单。

# 2、原理分析

看到客户端测试类中，我们只用了一行代码，就能完成对远程Rest服务的调用，相当的简单。为什么这么神奇，这几段代码是如何做到的呢？

## 2.1、@EnableFeignClients 注解声明客户端接口
入口是启动类上的注解```@EnableFeignClients```，源代码：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
	//basePackages的别名
	String[] value() default {};
	//声明基础包，spring boot启动后，会扫描该包下被@FeignClient注解的接口
	String[] basePackages() default {};
	//声明基础包的类，通过该类声明基础包
	Class<?>[] basePackageClasses() default {};
	//默认配置类
	Class<?>[] defaultConfiguration() default {};
	//直接声明的客户端接口类
	Class<?>[] clients() default {};
}
```
```@EnableFeignClients```的参数声明客户端接口的位置和默认的配置类。

## 2.2、@FeignClient注解，将接口声明为Feign客户端
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {
	// 要调用的服务的id，对应与eureka上注册的应用名
	// 带有可选协议前缀的服务的名称。name属性的同义词。
	// 必须为所有客户端指定一个name，无论是否提供url。
	// 可以指定为属性的key，例如:${propertyKey}。
	@AliasFor("name")
	String value() default "";
	
	// 如果存在contextId，contextId将代替name属性用作bean的name，但不会用作服务id。
	// 这个配置非常有用，如果不设置这个属性，那么@FeignClient注解的同1个name属性值不能出现在多个接口上
	// why? 因为不设置contextId属性的话，那么bean的名称就是name属性了，
	// 如果多个接口调用同1个服务（多个接口上的@FeignClient注解的的name属性值相同），那么就会出现相同名称的bean了，
	// 而spring默认是不允许bean覆盖的，当然你可以通过设置spring.bean.allowOverridde=true来允许bean覆盖，
	// 但这是一个隐患非常大的危险做法，强烈建议不要这么做。
	String contextId() default "";
	
	// 名称，对应与eureka上注册的应用名
	@AliasFor("value")
	String name() default "";
	
	// 如果有值，该值作为spring bean的qualifier， 否则使用contextId + 'FeignClient'作为qualifier
	// 另外contextId + 'FeignClient'中的contextId说明：当contextId存在使用contextId，不存在时依次使用serviceId、name、value
	String qualifier() default "";
	
	// http服务的url，绝对地址或相对地址，http协议是可选的
	String url() default "";
	
	// 是否解码404状态码的响应，如果不解码404响应则会抛出FeignExceptions异常
	boolean decode404() default false;
	
	// feign client的@Configuration配置类，可以包含重写feign配置的@Bean定义，比如Decoder、Encoder、Logger、Contract
	// 参阅FeignClientsConfiguration类查看默认的feign配置
	// 这里设置的配置类是Spring Configuration，将会在FeignContext中创建内部声明的Bean，用于不同的客户端进行隔离
	Class<?>[] configuration() default {};
	
	//声明hystrix调用失败后的方法
	Class<?> fallback() default void.class;
	
	// 为指定的feign client定义fallback工厂，这个工厂必须返回一个实现了标有@FeignClient注解的接口的降级策略实例
	// 该工厂必须是一个有效的spring bean
	Class<?> fallbackFactory() default void.class;
	
	// 所有方法级映射使用的路径前缀，是否想与@RibbonClient注解一起使用都可以
	String path() default "";
	
	// 是否将feign代理对象标记为首要bean，默认值为true。
	boolean primary() default true;
	
}
```

## 2.3、FeignClientsRegistrar 注册客户端

```@EnableFeignClients```注解上被注解了```@Import(FeignClientsRegistrar.class)```，```@Import```注解的作用是将指定的类作为Bean注入到Spring Context中，我们再来看被引入的```FeignClientsRegistrar```
```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
	
	private ResourceLoader resourceLoader;

	private Environment environment;
	
	...省略部分代码
	
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
	
	...省略部分代码
	
}
```

```FeignClientsRegistrar```类实现了3个接口:
- 接口```ResourceLoaderAware```用于注入```ResourceLoader```
- 接口```EnvironmentAware ```用于注入```Environment```
- 接口```ImportBeanDefinitionRegistrar```用于动态向Spring Context中注册bean

```ImportBeanDefinitionRegistrar```接口方法```registerBeanDefinitions```有两个参数
- AnnotationMetadata 包含被@Import注解类的信息
- BeanDefinitionRegistry bean定义注册中心

## 2.4、registerDefaultConfiguration方法，注册@EnableFeignClients注解的全局默认configuration
```java
private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
	// 获取@EnableFeignClients注解参数
	Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
	// 如果参数中包含defaultConfiguration默认配置
	if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
		String name;
		// 返回该类是否有上级类（即metadata注解的类是内部类/嵌套类、方法中的局部类、匿名类中的一种）
		// 如果此方法返回false，则表明metadata注解的类为顶级类。
		// 关于hasEnclosingClass方法的具体含义见：https://my.oschina.net/zhaopeng2012/blog/3146991
		if (metadata.hasEnclosingClass()) {
			// 不是顶级类，注解所标识类的上一级类的全限定类名
			name = "default." + metadata.getEnclosingClassName();
		}
		else {
			// 注解所标识的类的全限定类名
			name = "default." + metadata.getClassName();
		}
		// 注册客户端的配置Bean， 
		// 注意这里第3个参数值是@EnableFeignCliens注解的defaultConfiguration属性
		registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
	}
}
```

取出```@EnableFeignClients```注解的参数```defaultConfiguration```，动态注册到spring Context中。  

## 2.5、registerClientConfiguration 注册configuration

该方法用于注册feign配置，配置来源有2种：
- 1. ```@EnableFeignClients```注解的```defaultConfiguration```属性注册时（全局默认配置），name为default.xxx
- 2. ```@FeignClient```注解的```configuration```属性注册时（服务私有配置），name为contextId、name、value属性的值

有2个地方会调用该方法：
- ```FeignClientsRegistrar.registerDefaultConfiguration()```
-  ```FeignClientsRegistrar.registerFeignClients()```

**registerClientConfiguration** 方法源码如下：
```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name, Object configuration) {
	// 创建一个BeanDefinitionBuilder，注册bean的类为FeignClientSpecification
	BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(FeignClientSpecification.class);
	// 增加构造函数参数
	builder.addConstructorArgValue(name);
	// 关键代码：@EnableFeignCliens注解的defaultConfiguration属性
	builder.addConstructorArgValue(configuration);
	// 调用BeanDefinitionRegistry.registerBeanDefinition方法动态注册Bean
	// 这里的name有2种情况：
	// 1. @EnableFeignClients注解的defaultConfiguration属性注册时（全局默认配置），name为default.xxx
	// 2. @FeignClient注解的configuration属性注册时（服务私有配置），name为contextId、name、value属性的值
	registry.registerBeanDefinition(name + "." + FeignClientSpecification.class.getSimpleName(), builder.getBeanDefinition());
}
```
这里使用spring 动态注册bean的方式，注册了一个```FeignClientSpecification```的bean。

## 2.6、FeignClientSpecification 客户端定义
一个简单的pojo，继承了```NamedContextFactory.Specification```，两个属性```String name``` 和 ```Class<?>[] configuration```，用于```FeignContext```命名空间独立配置，后面会用到。
```java
class FeignClientSpecification implements NamedContextFactory.Specification {

	private String name;

	// 这个属性的值就是@EnableFeignClients的defaultConfiguration的值
	// 或者是@FeignClient注解的configuration属性的值
	private Class<?>[] configuration;
}
```

## 2.7、registerFeignClients方法，注册feign客户端
```java
public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
	// 生成一个scanner，扫描指定定包下的类
	ClassPathScanningCandidateComponentProvider scanner = getScanner();
	scanner.setResourceLoader(this.resourceLoader);

	Set<String> basePackages;
	// 获取@EnableFeignClients注解的所有属性
	Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
	// 包含@FeignClient注解的过滤器
	AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(FeignClient.class);
	// 获取@EnableFeignClients注解的clients属性的值
	final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
	
	if (clients == null || clients.length == 0) {
		//@EnableFeignClients没有声明clients，获取basePackages，设置过滤器（扫描包含@FeignClient注解的class）
		scanner.addIncludeFilter(annotationTypeFilter);
		// 根据value、basePackages、basePackageClasses获取要扫描的包
		basePackages = getBasePackages(metadata);
	}
	else {
		// @EnableFeignClients注解声明了clients
		final Set<String> clientClasses = new HashSet<>();
		basePackages = new HashSet<>();
		for (Class<?> clazz : clients) {
			// basePackages为声明的clients所在的包
			basePackages.add(ClassUtils.getPackageName(clazz));
			clientClasses.add(clazz.getCanonicalName());
		}
		// 增加过滤器，只包含声明的clients
		AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
			@Override
			protected boolean match(ClassMetadata metadata) {
				String cleaned = metadata.getClassName().replaceAll("\\$", ".");
				return clientClasses.contains(cleaned);
			}
		};
		scanner.addIncludeFilter(
			new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
	}
	
	// 遍历basePackages
	for (String basePackage : basePackages) {
		// 扫描包，根据过滤器找到候选的Bean
		Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
		// 遍历候选的bean
		for (BeanDefinition candidateComponent : candidateComponents) {
			// 判断候选bean是否是一个标有注解的bean
			if (candidateComponent instanceof AnnotatedBeanDefinition) {
				
				AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
				AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
				// 校验注解是否标识在接口上
				Assert.isTrue(annotationMetadata.isInterface(),
							  "@FeignClient can only be specified on an interface");

				// 获取@FeignClient注解的所有属性
				Map<String, Object> attributes = annotationMetadata.getAnnotationAttributes(
										FeignClient.class.getCanonicalName());
				// name获取顺序按优先级从高到低依次为contextId、value、name、serviceId。
				String name = getClientName(attributes);
				//注册@FeignClient的configuration属性的客户端配置
				registerClientConfiguration(registry, name,attributes.get("configuration"));
				//注册客户端
				registerFeignClient(registry, annotationMetadata, attributes);
			}
		}
	}
}
```
这个方法主要逻辑是扫描注解声明的客户端(标有```@FeignClient```的接口)，调用```registerFeignClient```方法注册到registry中。**这里是一个典型的spring动态注册bean的例子，可以参考这段代码在spring中轻松的实现类路径下class扫描，动态注册bean到spring中**。想了解spring类的扫描机制，可以断点到```ClassPathScanningCandidateComponentProvider.findCandidateComponents```方法中，一步步调试。

## 2.8、registerFeignClient方法，注册单个客户feign端bean
```java
private void registerFeignClient(BeanDefinitionRegistry registry,
				AnnotationMetadata annotationMetadata, 
				Map<String, Object> attributes) {
	
	String className = annotationMetadata.getClassName();
	// 构建一个FeignClientFactoryBean的bean工厂定义
	BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
	validate(attributes);
	//根据@FeignClient注解的参数，设置属性
	definition.addPropertyValue("url", getUrl(attributes));
	definition.addPropertyValue("path", getPath(attributes));
	String name = getName(attributes);
	definition.addPropertyValue("name", name);
	String contextId = getContextId(attributes);
	definition.addPropertyValue("contextId", contextId);
	definition.addPropertyValue("type", className);
	definition.addPropertyValue("decode404", attributes.get("decode404"));
	definition.addPropertyValue("fallback", attributes.get("fallback"));
	definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
	definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
	// bean的默认别名
	String alias = contextId + "FeignClient";
	AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

	boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
	// null
	// 相当于@Primary注解的作用
	beanDefinition.setPrimary(primary);
	//获取@FeignClient注解中指定的qualifier属性
	String qualifier = getQualifier(attributes);
	if (StringUtils.hasText(qualifier)) {
		// 如果用户自己指定了qualifier属性，则使用注解中的qualifier属性作为bean的别名
		// 否则使用上述默认的 contextId + "FeignClient";
		alias = qualifier;
	}

	// 注册，这里为了简写，新建一个BeanDefinitionHolder，调用BeanDefinitionReaderUtils静态方法注册
	BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, new String[] { alias });
	BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}

/**
* 获取服务名， 获取顺序为serviceId、name、value
*/
String getName(Map<String, Object> attributes) {
	String name = (String) attributes.get("serviceId");
	if (!StringUtils.hasText(name)) {
		name = (String) attributes.get("name");
	}
	if (!StringUtils.hasText(name)) {
		name = (String) attributes.get("value");
	}
	name = resolve(name);
	return getName(name);
}

/**
* 获取contextId， 获取顺序为contextId、serviceId、name、value
*
* 如果存在contextId，contextId将代替name属性用作bean的name，但不会用作服务id。
* 这个配置非常有用，如果不设置这个属性，那么@FeignClient注解的同1个name属性值不能出现在多个接口上
* why? 因为不设置contextId属性的话，那么bean的名称就是name属性了，
* 如果多个接口调用同1个服务（多个接口上的@FeignClient注解的的name属性值相同），那么就会出现相同名称的bean了，
* 而spring默认是不允许bean覆盖的，当然你可以通过设置spring.bean.allowOverridde=true来允许bean覆盖，
* 但这是一个隐患非常大的危险做法，强烈建议不要这么做。
*
* 看到这里的getContextId方法就应该知道contextId与name的区别了吧
*/
private String getContextId(Map<String, Object> attributes) {
	String contextId = (String) attributes.get("contextId");
	if (!StringUtils.hasText(contextId)) {
		// 未设置contextId属性时，使用serviceId、name、value
		return getName(attributes);
	}

	contextId = resolve(contextId);
	// 验证contextId是否是有效的URI
	return getName(contextId);
}

// @FeignClinet注解的属性(contextId、serviceId、name、value)支持${keyProperty}环境变量
private String resolve(String value) {
	if (StringUtils.hasText(value)) {
		return this.environment.resolvePlaceholders(value);
	}
	return value;
}

// 验证http[s]://name是否是1个有效的URI
static String getName(String name) {
	if (!StringUtils.hasText(name)) {
		return "";
	}

	String host = null;
	try {
		String url;
		if (!name.startsWith("http://") && !name.startsWith("https://")) {
			url = "http://" + name;
		}
		else {
			url = name;
		}
		host = new URI(url).getHost();

	}
	catch (URISyntaxException e) {
	}
	Assert.state(host != null, "Service id not legal hostname (" + name + ")");
	return name;
}
```

```registerFeignClient```方法主要是将```FeignClientFactoryBean```工厂```Bean```注册到registry中，spring初始化后，会调用```FeignClientFactoryBean```的getObject方法创建bean注册到spring context中。

## 2.9、FeignClientFactoryBean 创建feign客户端的工厂
```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
	private Class<?> type;

	private String name;

	private String url;

	private String contextId;

	private String path;

	private boolean decode404;

	private ApplicationContext applicationContext;

	private Class<?> fallback = void.class;

	private Class<?> fallbackFactory = void.class;
	
	...省略部分代码
}
```
```FeignClientFactoryBean```实现了```FactoryBean```接口，是一个工厂bean

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191229142123.png

### 2.9.1、FeignClientFactoryBean.getObject方法

```java
@Override
public Object getObject() throws Exception {
	return getTarget();
}

<T> T getTarget() {
	// FeignContext在FeignAutoConfiguration中自动注册，FeignContext用于客户端配置类独立注册，后面具体分析
	// 这行代码会导致FeignAutoConfiguration中的FeignContext创建
	FeignContext context = this.applicationContext.getBean(FeignContext.class);
	// 创建Feign.Builder对象
	Feign.Builder builder = feign(context);

	// 如果@FeignClient注解没有设置url参数
	if (!StringUtils.hasText(this.url)) {
		// url为@FeignClient注解的name参数
		if (!this.name.startsWith("http")) {
			// 例如 http://md-admin-web
			this.url = "http://" + this.name;
		}
		else {
			// @FeignClient注解的name属性以http开头的
			this.url = this.name;
		}
		//加上path
		this.url += cleanPath();
		// 返回loadBlance客户端，也就是ribbon+eureka的客户端
		return (T) loadBalance(builder, context,
							   new HardCodedTarget<>(this.type, this.name, this.url));
	}
	
	//@FeignClient设置了url参数，不做负载均衡
	if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
		this.url = "http://" + this.url;
	}
	//加上path
	String url = this.url + cleanPath();
	//从FeignContext中获取client
	Client client = getOptional(context, Client.class);
	if (client != null) {
		if (client instanceof LoadBalancerFeignClient) {
			// 有url参数，不做负载均衡，但是客户端是ribbon，或者实际的客户端
			client = ((LoadBalancerFeignClient) client).getDelegate();
		}
		builder.client(client);
	}
	// 从FeignContext中获取Targeter
	Targeter targeter = get(context, Targeter.class);
	// 生成客户端代理
	return (T) targeter.target(this, builder, context, new HardCodedTarget<>(this.type, this.name, url));
}
```
这段代码有个比较重要的逻辑，如果在```@FeignClient```注解中设置了url参数，就不走Ribbon，直接url调用，否则通过Ribbon调用，实现客户端负载均衡。

可以看到，生成Feign客户端所需要的各种配置对象，都是通过```FeignContex```中获取的。

### 2.9.2、FeignContext 隔离配置

在```@FeignClient```注解参数```configuration```，指定的类是Spring的Configuration Bean，里面方法上加```@Bean```注解实现Bean的注入，可以指定feign客户端的各种配置，包括```Encoder/Decoder/Contract/Feign.Builder```等。**不同的客户端指定不同配置类，就需要对配置类进行隔离**，```FeignContext```就是用于隔离配置的。

```java
public class FeignContext extends NamedContextFactory<FeignClientSpecification> {
	public FeignContext() {
		super(FeignClientsConfiguration.class, "feign", "feign.client.name");
	}
}
```
```FeignContext```继承```NamedContextFactory```，空参数构造函数指定```FeignClientsConfiguration```类为默认配置。
```NamedContextFactory```实现接口```ApplicationContextAware```，注入```ApplicationContextAware```作为parent：

```java
/**
* 这里的泛型C是FeignClientSpecification
*/
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
	private final String propertySourceName;
	private final String propertyName;
	
	//命名空间对应的Spring Context
	private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();
	//不同命名空间的定义
	private Map<String, C> configurations = new ConcurrentHashMap<>();

	private ApplicationContext parent;
	//默认配置类
	private Class<?> defaultConfigType;
	
	// 设置配置，在FeignAutoConfiguration中将Spring Context中的所有FeignClientSpecification设置进来
	// 1. 如果@EnableFeignClients有设置参数defaultConfiguration也会加进来，前面已经分析
	// 在registerDefaultConfiguration方法中注册的FeignClientSpecification Bean
	// 2. @FeignClient注解的configuration属性配置也会加进来
	// 在registerFeignClients()方法中注册的FeignClientSpecification Bean
	public void setConfigurations(List<C> configurations) {
		for (C client : configurations) {
			this.configurations.put(client.getName(), client);
		}
	}

	// 获取指定命名空间的ApplicationContext，先从缓存中获取，没有就创建
	protected AnnotationConfigApplicationContext getContext(String name) {
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}

	/**
	* 合并默认配置(default.xxx)和私有配置(name对应的配置)
	*/
	protected AnnotationConfigApplicationContext createContext(String name) {
		// 新建AnnotationConfigApplicationContext
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		// 根据name在configurations找到所有的配置类，注册到context中
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
		// 将default.开头的默认配置也注册到Context中
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
		// 注册默认配置bean， 对于feign而言就是FeignContext的构造方法中传入的FeignClientsConfiguration
		context.register(PropertyPlaceholderAutoConfiguration.class, this.defaultConfigType);
		// 注册解析feign.client.name配置的MapPropertySource bean
		context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
				this.propertySourceName,
				Collections.<String, Object>singletonMap(this.propertyName, name)));
		// 设置parent
		if (this.parent != null) {
			// Uses Environment from parent as well as beans
			context.setParent(this.parent);
			// jdk11 issue
			// https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
			context.setClassLoader(this.parent.getClassLoader());
		}
		context.setDisplayName(generateDisplayName(name));
		// 刷新，完成bean生成
		context.refresh();
		return context;	
	}
	
	// 从命名空间中获取指定类型的Bean
	public <T> T getInstance(String name, Class<T> type) {
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return context.getBean(type);
		}
		return null;
	}
	
	// 从命名空间中获取指定类型的Bean
	public <T> Map<String, T> getInstances(String name, Class<T> type) {
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return BeanFactoryUtils.beansOfTypeIncludingAncestors(context, type);
		}
		return null;
	}
}
```

关键的方法是```createContext```，为每个命名空间独立创建的```ApplicationContext```，设置parent为外部传入的Context，这样就可以共用外部的Context中的Bean，又有各种独立的配置Bean，熟悉springMVC的同学应该知道，springMVC中创建的```WebApplicatonContext```里面也有个parent，原理跟这个类似。

从```FeignContext```中获取Bean，需要传入命名空间，根据命名空间找到缓存中的```ApplicationContext```，先从自己注册的```Bean```中获取bean，没有获取到再从到parent中获取。

### 2.9.3、创建Feign.Builder
了解了```FeignContext```的原理，我们再来看feign最重要的构建类创建过程

在```FeignClientFactoryBean```的```getTarget```方法中调用了```feign()```方法返回了```Feign.Builder```对象：
```java
class FeignClientFactoryBean
	implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {

	<T> T getTarget() {
		Feign.Builder builder = feign(context);
	}
	
	protected Feign.Builder feign(FeignContext context) {
		// 从FeignContext中获取注册的FeignLoggerFactory日志工厂bean
		FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
		// 创建feign的Logger接口的日志实现bean
		Logger logger = loggerFactory.create(this.type);

		// 从FeignContext中获取注册的Feign.Builder bean，设置Encoder/Decoder/Contract
		Feign.Builder builder = get(context, Feign.Builder.class)
			// 这里的都是必须的配置， 
			// 需要注意的是这里的get(context, xx.class)都是先从自身命名空间上下文中找bean（私有配置），
			// 如果找不到会从父上下文中找bean使用全局配置, 这就是为什么@FeignClient的configuration属性可以没有的原因。
			
			// 配置日志实现，默认Slf4jLogger
			.logger(logger)
			// 配置form、bdoy参数编码器, 默认SpringEncoder
			.encoder(get(context, Encoder.class)) 
			 // 配置响应解码器,默认OptionalDecoder(持有SpringDecoder)
			.decoder(get(context, Decoder.class))
			// 配置契约实现SpringMvcContract
			.contract(get(context, Contract.class)); 

		// 配置feign的其他可选参数
		configureFeign(context, builder);

		return builder;
	}
	
	// 配置feign其他参数
	protected void configureFeign(FeignContext context, Feign.Builder builder) {
		FeignClientProperties properties = this.applicationContext.getBean(FeignClientProperties.class);
		if (properties != null) {
			// 判断client的独立可选配置是否可以覆盖全局的可选配置
			if (properties.isDefaultToProperties()) {
				// 配置可选参数，全局的可选配置可以被client独立的配置覆盖
				// 配置可选配置（日志级别、拦截器、重试、查询参数编码器、错误解码器、是否解码404响应）
				configureUsingConfiguration(context, builder);
				
				// 使用feign.client.config.default的默认配置
				configureUsingProperties(properties.getConfig().get(properties.getDefaultConfig()), builder);
				
				// 使用feign.client.config.{contextId}的feign client独立的配置，
				// 这样达到了默认使用全局配置，每个client又可以覆盖全局默认配置，实现配置个性化的目的
				configureUsingProperties(properties.getConfig().get(this.contextId), builder);
			}
			else {
				// 使用feign.client.config.default的默认配置
				configureUsingProperties(properties.getConfig().get(properties.getDefaultConfig()), builder);
				
				// 使用feign.client.config.{contextId}的feign client独立的配置，
				// 这样达到了默认使用全局配置，每个client又可以覆盖全局默认配置，实现配置个性化的目的
				configureUsingProperties(properties.getConfig().get(this.contextId), builder);
				
				// 配置可选参数，全局的可选配置覆盖client独立的配置
				// 配置可选配置（日志级别、拦截器、重试、查询参数编码器、错误解码器、是否解码404响应）
				configureUsingConfiguration(context, builder);
			}
		}
		else {
			// 配置可选配置（日志级别、拦截器、重试、查询参数编码器、错误解码器、是否解码404响应）
			configureUsingConfiguration(context, builder);
		}
	}
	
	protected void configureUsingConfiguration(FeignContext context, Feign.Builder builder) {
		// 获取可选配置：日志级别
		Logger.Level level = getOptional(context, Logger.Level.class);
		if (level != null) {
			// 设置日志级别
			builder.logLevel(level);
		}
		// 获取可选配置：重试条件
		Retryer retryer = getOptional(context, Retryer.class);
		if (retryer != null) {
			// 设置重试条件
			builder.retryer(retryer);
		}
		
		// 获取可选配置：错误解码器（非2xx、不解码404时的使用它将响应数据翻译为相应的异常抛出）
		ErrorDecoder errorDecoder = getOptional(context, ErrorDecoder.class);
		if (errorDecoder != null) {
			// 设置错误解码器
			builder.errorDecoder(errorDecoder);
		}
		
		// 获取可选配置：http请求相关参数(connectTimeout、readTimeout等，
		// 它会传递给具体的发送http请求的client， 例如ApacheHttpClient、OkHttpClient等)
		Request.Options options = getOptional(context, Request.Options.class);
		if (options != null) {
			// 设置http请求相关参数
			builder.options(options);
		}
		
		// 获取可选配置：发送请求前的拦截器
		Map<String, RequestInterceptor> requestInterceptors = context.getInstances(this.contextId, RequestInterceptor.class);
		if (requestInterceptors != null) {
			// 设置请求拦截器
			builder.requestInterceptors(requestInterceptors.values());
		}
		
		// 获取可选配置：查询参数编码器
		// 用于将方法参数的java bean、Map对象序列化为http请求url的?后的查询参数，如a=b&c=d
		QueryMapEncoder queryMapEncoder = getOptional(context, QueryMapEncoder.class);
		if (queryMapEncoder != null) {
			// 设置查询参数编码器
			builder.queryMapEncoder(queryMapEncoder);
		}
		//是否解码404响应
		if (this.decode404) {
			// 设置当响应状态码为404时是否使用decoder解码response，
			// 若该参数为false，则会使用errorDecoder翻译为相应的异常抛出
			builder.decode404();
		}
	}
}
```

feign()方法设置了```Feign.Builder```所必须的参数``Encoder```/```Decoder```/```Contract```，其他参数都是可选的。这三个必须的参数从哪里来的呢？答案是在```FeignContext```的构造器中，传入了默认的配置```FeignClientsConfiguration```，这个配置类里面初始化了这三个参数。

```java
@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {
	@Autowired
	private ObjectFactory<HttpMessageConverters> messageConverters;

	@Autowired(required = false)
	private List<AnnotatedParameterProcessor> parameterProcessors = new ArrayList<>();

	@Autowired(required = false)
	private List<FeignFormatterRegistrar> feignFormatterRegistrars = new ArrayList<>();

	@Autowired(required = false)
	private Logger logger;

	@Autowired(required = false)
	private SpringDataWebProperties springDataWebProperties;

	// Decoder bean，默认通过HttpMessageConverters进行处理
	@Bean
	@ConditionalOnMissingBean
	public Decoder feignDecoder() {
		return new OptionalDecoder(
			new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
	}

	// Encoder bean，默认通过HttpMessageConverters进行处理
	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
	public Encoder feignEncoder() {
		return new SpringEncoder(this.messageConverters);
	}

	// Encoder bean
	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Pageable")
	@ConditionalOnMissingBean
	public Encoder feignEncoderPageable() {
		PageableSpringEncoder encoder = new PageableSpringEncoder(
			new SpringEncoder(this.messageConverters));
		if (springDataWebProperties != null) {
			encoder.setPageParameter(
				springDataWebProperties.getPageable().getPageParameter());
			encoder.setSizeParameter(
				springDataWebProperties.getPageable().getSizeParameter());
			encoder.setSortParameter(
				springDataWebProperties.getSort().getSortParameter());
		}
		return encoder;
	}

	// Contract bean，通过SpringMvcContract进行处理接口
	@Bean
	@ConditionalOnMissingBean
	public Contract feignContract(ConversionService feignConversionService) {
		return new SpringMvcContract(this.parameterProcessors, feignConversionService);
	}

	@Bean
	public FormattingConversionService feignConversionService() {
		FormattingConversionService conversionService = new DefaultFormattingConversionService();
		for (FeignFormatterRegistrar feignFormatterRegistrar : this.feignFormatterRegistrars) {
			feignFormatterRegistrar.registerFormatters(conversionService);
		}
		return conversionService;
	}

	//默认不重试
	@Bean
	@ConditionalOnMissingBean
	public Retryer feignRetryer() {
		return Retryer.NEVER_RETRY;
	}

	// 默认的builder
	@Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	public Feign.Builder feignBuilder(Retryer retryer) {
		return Feign.builder().retryer(retryer);
	}

	// 日志工厂
	@Bean
	@ConditionalOnMissingBean(FeignLoggerFactory.class)
	public FeignLoggerFactory feignLoggerFactory() {
		return new DefaultFeignLoggerFactory(this.logger);
	}
	
	// jackson处理spring data的Page对象
	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Page")
	public Module pageJacksonModule() {
		return new PageJacksonModule();
	}
	
	// 存在hystrix时，hystrix自动注入
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {
		
		// HystrixFeign的builder，全局关掉Hystrix配置feign.hystrix.enabled=false
		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}

	}
}
```

可以看到，feign需要的decoder/enoder通过适配器共用springMVC中的```HttpMessageConverters```引入。

feign有自己的注解体系，这里通过```SpringMvcContract```适配了springMVC的注解体系。

### 2.9.4、SpringMvcContract 适配feign注解体系

SpringMvcContract继承了feign的类```Contract.BaseContract```，作用是解析接口方法上的注解和方法参数，生成```MethodMetadata```用于接口方法调用过程中组装http请求。
```java
public class SpringMvcContract extends Contract.BaseContract implements ResourceLoaderAware {
	
	/**
	* 
	*/
	@Override
	public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
		this.processedMethods.put(Feign.configKey(targetType, method), method);
		
		// 调用父类Contract.BaseContract的处理逻辑
		// 而父类处理过程中会调用processAnnotationOnClass、processAnnotationOnMethod、processAnnotationsOnParameter
		// 这3个方法来解析类、方法、参数上的注解，SpringMvcContract通过重写了这3个方法解析spring mvc自己的
		// @RequestMapping、@RequestParam、@RequestBody、@GetMapping...等等注解
		// 来实现利用spring这套解决方案实现微服务间互相调用的闭环
		MethodMetadata md = super.parseAndValidateMetadata(targetType, method);

		RequestMapping classAnnotation = findMergedAnnotation(targetType, RequestMapping.class);
		if (classAnnotation != null) {
			// produces - use from class annotation only if method has not specified this
			if (!md.template().headers().containsKey(ACCEPT)) {
				parseProduces(md, method, classAnnotation);
			}

			// consumes -- use from class annotation only if method has not specified this
			if (!md.template().headers().containsKey(CONTENT_TYPE)) {
				parseConsumes(md, method, classAnnotation);
			}

			// headers -- class annotation is inherited to methods, always write these if
			// present
			parseHeaders(md, method, classAnnotation);
		}
		return md;
	}

	/**
	* 处理class上的注解
	*/
	@Override
	protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
		...省略代码
	}
	
	/**
	* 处理方法上的注解：@RequestMapping、
	*/
	@Override
	protected void processAnnotationOnMethod(MethodMetadata data,
			Annotation methodAnnotation, Method method) {
		...省略代码
	}
	
	/**
	* 处理参数上的注解
	*/
	protected boolean processAnnotationsOnParameter(MethodMetadata data,
			Annotation[] annotations, int paramIndex) {
		...省略代码
	}
}
```
几个覆盖方法分别是处理类上的注解，处理方法，处理方法上的注解，处理方法参数注解，最终生成完整的```MethodMetadata```。feign自己提供的Contract和扩展javax.ws.rx的Contract原理都是类似的。


## 2.9.5、FeignAutoConfiguration
```Feign.Builder```生成后，就要用Target生成feign客户端的动态代理，这里```FeignClientFactoryBean```中使用```Targeter```，Targeter有两个实现类，分别是```HystrixTargeter```和```DefaultTargeter```，那么默认的Targeter又是怎么来的呢？

在spring-cloud-openfeign-core项目的```META-INF\spring.factories```文件中有```FeignAutoConfiguration```的自动化配置，关于spring.factories自动化配置的原理见[springboot2.2自动注入文件spring.factories如何加载详解](https://my.oschina.net/zhaopeng2012/blog/3144983 "springboot2.2自动注入文件spring.factories如何加载详解")
- **spring.factories**
```peoperties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
org.springframework.cloud.openfeign.hateoas.FeignHalAutoConfiguration,\
org.springframework.cloud.openfeign.FeignAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.loadbalancer.FeignLoadBalancerAutoConfiguration
```

- **FeignAutoConfiguration**
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Feign.class)
@EnableConfigurationProperties({ FeignClientProperties.class, FeignHttpClientProperties.class })
@Import(DefaultGzipDecoderConfiguration.class)
public class FeignAutoConfiguration {
	/**
 	* FeignClientSpecification包含了feign配置类，配置来源有2种：
	* 1. @EnableFeignClients注解的defaultConfiguration属性注册时（全局默认配置），name为default.xxx
	* 2. @FeignClient注解的configuration属性注册时（服务私有配置），name为contextId、name、value属性的值
	*
	* 有2个地方会调用该方法产生FeignClientSpecification：
	* 1. FeignClientsRegistrar.registerDefaultConfiguration()
	* 2. FeignClientsRegistrar.registerFeignClients()
	*/
	@Autowired(required = false)
	private List<FeignClientSpecification> configurations = new ArrayList<>();

	/**
	* spring cloud commons项目中的HasFeatures，表示使用了Feign特性，用于spring boot actuator监控
	*/
	@Bean
	public HasFeatures feignFeature() {
		return HasFeatures.namedFeature("Feign", Feign.class);
	}

	/**
	* 关键代码：feign客户端配置隔离的实现
	* 创建FeignClientSpecification， 每个feign客户端能够有各自私有的configuration就靠它了，
	* 他是命名空间隔离的applicationContext， 每个feign客户端都有自己的applicationContext
	*/
	@Bean
	public FeignContext feignContext() {
		FeignContext context = new FeignContext();
		// 一定要注意这里把所有的配置都设置进去了
		// 包括@EnableFeignClients的全局defaultConfiguration和@FeignClient的私有configuration配置
		// FeignClientFactoryBean类中getTarget创建Feign.Bulider的get(context, beanType)都是从它里面取的
		// feign客户端自身的applicationContext取不到就到父applicationContext中去取，
		// 达到先取私有配置bean(例如encoder、client、logger、decoder、loggerLevel)，
		// 私有配置取不到再取全局配置bean的功能
		context.setConfigurations(this.configurations);
		return context;
	}

	/**
	* 类路径中存在HystrixFeign这个类，使用hystrix实现的Targeter
	*/
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
	protected static class HystrixFeignTargeterConfiguration {
		/**
		* 实现了熔断降级功能的Targeter
		*/
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new HystrixTargeter();
		}

	}

	/**
	* 类路径中没有HystrixFeign这个类，使用默认实现DefaultTargeter
	*/
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
	protected static class DefaultFeignTargeterConfiguration {
		
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new DefaultTargeter();
		}

	}
	
	/**
	* 引入了feign-httpclient.jar，使用apache HttpClient发送请求
	*/
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(ApacheHttpClient.class)
	@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
	@ConditionalOnMissingBean(CloseableHttpClient.class)
	@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
	protected static class HttpClientFeignConfiguration {
		@Bean
		@ConditionalOnMissingBean(Client.class)
		public Client feignClient(HttpClient httpClient) {
			return new ApacheHttpClient(httpClient);
		}
	}
	
	/**
	* 引入了feign-okhttp.jar， 使用okhttp发送请求
	*/
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(OkHttpClient.class)
	@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
	@ConditionalOnMissingBean(okhttp3.OkHttpClient.class)
	@ConditionalOnProperty("feign.okhttp.enabled")
	protected static class OkHttpFeignConfiguration {
		@Bean
		public okhttp3.OkHttpClient client(OkHttpClientFactory httpClientFactory,
				ConnectionPool connectionPool,
				FeignHttpClientProperties httpClientProperties) {
			Boolean followRedirects = httpClientProperties.isFollowRedirects();
			Integer connectTimeout = httpClientProperties.getConnectionTimeout();
			Boolean disableSslValidation = httpClientProperties.isDisableSslValidation();
			this.okHttpClient = httpClientFactory.createBuilder(disableSslValidation)
					.connectTimeout(connectTimeout, TimeUnit.MILLISECONDS)
					.followRedirects(followRedirects).connectionPool(connectionPool)
					.build();
			return this.okHttpClient;
		}
	}
}
```


- **@FeignClient配置隔离体系总结**

需要注意的是```FeignAutoConfiguration```中的bean都是默认的全局配置， 而每个```@FeignClient```都可以有其私有的configuration属性配置，在```FeignClientFactoryBean.feign()```方法中创建```Feign.builder().enocder(xx).decoder(xxx).targeter()```时调用的
```get(context, xx.class)```方法都是先从自身命名空间上下文中找bean（私有配置），如果找不到会从父上下文中找bean使用全局配置, 这就是为什么```@FeignClient的configuration```属性可以没有的原因。


### 2.9.6、Targeter 生成接口动态代理
- **DefaultTargeter**

```DefaultTargeter```很简单，直接调用```HardCodedTarget```生成动态代理，```HystrixTargeter```源码如下：
```java
class DefaultTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
		// 直接由feign自身的实现产生jdk 动态代理工厂产生代理对象
		return feign.target(target);
	}

}
```

- **HystrixTargeter**
```java
class HystrixTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
		// 如果不是HystrixFeign.Builder，直接调用target生成代理
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		
	
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		// 优先使用contextId属性,  如果contextId为空则使用name属性
		String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
				: factory.getContextId();
		// context是配置隔离的applicationContext
		// 优先取client私有配置中的SetterFactory，若未配置私有的，则取全局默认的
		SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
		// 私有的或全局的配置了SetterFactory
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
		
		//找到fallback或者fallbackFactory，设置到hystrix中
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(name, context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(name, context, target, builder,
					fallbackFactory);
		}

		// 无降级策略，直接调用target生成代理
		return feign.target(target);
	}
	
	private <T> T targetWithFallbackFactory(String feignClientName, 
							FeignContext context,
							Target.HardCodedTarget<T> target, 
							HystrixFeign.Builder builder,
							Class<?> fallbackFactoryClass) {
		
		FallbackFactory<? extends T> fallbackFactory = (FallbackFactory<? extends T>) getFromContext(
			"fallbackFactory", feignClientName, context, fallbackFactoryClass,
			FallbackFactory.class);
		return builder.target(target, fallbackFactory);
	}

	private <T> T targetWithFallback(String feignClientName, 
					FeignContext context,
					Target.HardCodedTarget<T> target, 
					HystrixFeign.Builder builder,
					Class<?> fallback) {
		
		T fallbackInstance = getFromContext("fallback", feignClientName, context, fallback, target.type());
		return builder.target(target, fallbackInstance);
	}
}
```
- HystrixFeign.Builder
```java
public final class HystrixFeign {
	public static final class Builder extends feign.Feign.Builder {
		private Contract contract = new Default();
		private SetterFactory setterFactory = new feign.hystrix.SetterFactory.Default();
		
		public <T> T target(Target<T> target, T fallback) {
			return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null).newInstance(target);
		}

		public <T> T target(Target<T> target, FallbackFactory<? extends T> fallbackFactory) {
			return build(fallbackFactory).newInstance(target);
		}
		
		Feign build(final FallbackFactory<?> nullableFallbackFactory) {
			// 替换掉Feign.Bulider中默认的
			// private InvocationHandlerFactory invocationHandlerFactory = new InvocationHandlerFactory.Default();
			// 由此可知当hystrix启用时feign客户端的动态代理实现是在HystrixInvocationHandler中
			// 也就是说我们在contrller层调用feign客户端(例如UserFeignClient)时会进入到HystrixInvocationHandler中
			// 拦截方法调用
			super.invocationHandlerFactory(new InvocationHandlerFactory() {
				@Override
				public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
					return new HystrixInvocationHandler(target, dispatch, setterFactory, nullableFallbackFactory);
				}
			});
			
			super.contract(new HystrixDelegatingContract(contract));
			return super.build();
		}
	}
}
```

- HystrixInvocationHandler
```java
final class HystrixInvocationHandler implements InvocationHandler {
	private final Target<?> target;
	private final Map<Method, MethodHandler> dispatch;
	private final FallbackFactory<?> fallbackFactory; // Nullable
	private final Map<Method, Method> fallbackMethodMap;
	private final Map<Method, Setter> setterMethodMap;

	@Override
	public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
		...省略部分代码
			
		HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
				@Override
				protected Object run() throws Exception {
					try {
						return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
					} catch (Exception e) {
						throw e;
					} catch (Throwable t) {
						throw (Error) t;
					}
				}

				@Override
				protected Object getFallback() {
					if (fallbackFactory == null) {
						return super.getFallback();
					}
					try {
						Object fallback = fallbackFactory.create(getExecutionException());
						Object result = fallbackMethodMap.get(method).invoke(fallback, args);
						if (isReturnsHystrixCommand(method)) {
							return ((HystrixCommand) result).execute();
						} else if (isReturnsObservable(method)) {
							// Create a cold Observable
							return ((Observable) result).toBlocking().first();
						} else if (isReturnsSingle(method)) {
							// Create a cold Observable as a Single
							return ((Single) result).toObservable().toBlocking().first();
						} else if (isReturnsCompletable(method)) {
							((Completable) result).await();
							return null;
						} else if (isReturnsCompletableFuture(method)) {
							return ((Future) result).get();
						} else {
							return result;
						}
					} catch (IllegalAccessException e) {
						// shouldn't happen as method is public due to being an interface
						throw new AssertionError(e);
					} catch (InvocationTargetException | ExecutionException e) {
						// Exceptions on fallback are tossed by Hystrix
						throw new AssertionError(e.getCause());
					} catch (InterruptedException e) {
						// Exceptions on fallback are tossed by Hystrix
						Thread.currentThread().interrupt();
						throw new AssertionError(e.getCause());
					}
				}
		};
		
		// 是jdk8的接口中的default方法，直接执行
		if (Util.isDefault(method)) {
			return hystrixCommand.execute();
		}
		// 返回值是HystrixCommand类型，不真正执行，直接返回上面创建的hystrixCommand
		else if (isReturnsHystrixCommand(method)) {
			return hystrixCommand;
		}
		//返回值类型是Observable
		else if (isReturnsObservable(method)) {
			// Create a cold Observable
			return hystrixCommand.toObservable();
		}
		// 返回值类型是Single
		else if (isReturnsSingle(method)) {
			// Create a cold Observable as a Single
			return hystrixCommand.toObservable().toSingle();
		} 
		// 返回值类型是Completable
		else if (isReturnsCompletable(method)) {
			return hystrixCommand.toObservable().toCompletable();
		}
		// 返回值类型是CompletableFuture
		else if (isReturnsCompletableFuture(method)) {
			return new ObservableCompletableFuture<>(hystrixCommand);
		}
		// 其他情况直接执行
		return hystrixCommand.execute();
		
	}
	
	private boolean isReturnsCompletable(Method method) {
		return Completable.class.isAssignableFrom(method.getReturnType());
	}

	private boolean isReturnsHystrixCommand(Method method) {
		return HystrixCommand.class.isAssignableFrom(method.getReturnType());
	}

	private boolean isReturnsObservable(Method method) {
		return Observable.class.isAssignableFrom(method.getReturnType());
	}

	private boolean isReturnsCompletableFuture(Method method) {
		return CompletableFuture.class.isAssignableFrom(method.getReturnType());
	}

	private boolean isReturnsSingle(Method method) {
		return Single.class.isAssignableFrom(method.getReturnType());
	}

}
```

## 3、loadBalance方法，客户端负载均衡
### 3.1、loadBalance方法的调用
如果```@FeignClient```注解中没有配置url参数，将会通过```loadBalance```方法生成Ribbon的动态代理：
```java
class FeignClientFactoryBean
	implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
	
	private String name;

	private String url;

	private String contextId;
	
	...省略部分代码
		
	@Override
	public Object getObject() throws Exception {
		return getTarget();
	}
		
	<T> T getTarget() {
		...省略部分代码
			
		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		
		...省略部分代码
	}
		
	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
								HardCodedTarget<T> target) {
		// 这里获取到的Client是LoadBalancerFeignClient
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
			"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
	
	...省略部分代码
	
}
```

### 3.2、FeignRibbonClientAutoConfiguration

之前我们我们已经分析过在spring-cloud-openfeign-core项目的```META-INF\spring.factories```文件中有```FeignAutoConfiguration```的自动化配置，这里我们可以看到```spring.factories```文件里面还有个```FeignRibbonClientAutoConfiguration```自动化配置类，从名称就可以知道它与负载均衡相关。

- **spring.factories**
```peoperties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
org.springframework.cloud.openfeign.hateoas.FeignHalAutoConfiguration,\
org.springframework.cloud.openfeign.FeignAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.loadbalancer.FeignLoadBalancerAutoConfiguration
```

- **FeignRibbonClientAutoConfiguration**
```java
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@ConditionalOnProperty(value = "spring.cloud.loadbalancer.ribbon.enabled",
		matchIfMissing = true)
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(FeignAutoConfiguration.class)
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
// Order is important here, last should be the default, first should be optional
// see
// https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {
}
```
可以看到它引入了HttpClientFeignLoadBalancedConfiguration、OkHttpFeignLoadBalancedConfiguration、DefaultFeignLoadBalancedConfiguration这3个类，我们接下来先看不引入apache http client、okhttp这些http库时的执行流程

- LoadBalancerFeignClient的创建
```java
@Configuration(proxyBeanMethods = false)
class DefaultFeignLoadBalancedConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
			SpringClientFactory clientFactory) {
		return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
				clientFactory);
	}

}
```

- LoadBalancerFeignClient
```java
public class LoadBalancerFeignClient implements Client {

	static final Request.Options DEFAULT_OPTIONS = new Request.Options();

	private final Client delegate;

	private CachingSpringLoadBalancerFactory lbClientFactory;

	private SpringClientFactory clientFactory;

	/**
	* @param request 是feign自身的Request对象，持有发送http请求的Client实现
	* @param options  http请求的相关配置(connectTimeout、readTimeout)
	*/
	@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			//获取URI
			URI asUri = URI.create(request.url());
			//获取客户端的名称
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			//创建RibbonRequest
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
				this.delegate, request, uriWithoutHost);
			//配置
			IClientConfig requestConfig = getClientConfig(options, clientName);
			//获取FeignLoadBalancer，发请求，转换Response
			return lbClient(clientName)
				.executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
	
	private FeignLoadBalancer lbClient(String clientName) {
		return this.lbClientFactory.create(clientName);
	}

}
```

- CachingSpringLoadBalancerFactory
```java
public class CachingSpringLoadBalancerFactory {
	protected final SpringClientFactory factory;

	protected LoadBalancedRetryFactory loadBalancedRetryFactory = null;

	private volatile Map<String, FeignLoadBalancer> cache = new ConcurrentReferenceHashMap<>();


	public FeignLoadBalancer create(String clientName) {
		// 先从缓存获取，缓存中没有再创建
		FeignLoadBalancer client = this.cache.get(clientName);
		if (client != null) {
			return client;
		}
		IClientConfig config = this.factory.getClientConfig(clientName);
		// 取出负载均衡实现
		ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
		ServerIntrospector serverIntrospector = this.factory.getInstance(clientName, ServerIntrospector.class);
		// 创建FeignLoadBalancer或带有重试功能的RetryableFeignLoadBalancer
		client = this.loadBalancedRetryFactory != null
			? new RetryableFeignLoadBalancer(lb, config, serverIntrospector, this.loadBalancedRetryFactory)
			: new FeignLoadBalancer(lb, config, serverIntrospector);
		// 放到内存缓存中
		this.cache.put(clientName, client);
		return client;
	}
}
```

- FeignLoadBalancer
```java
public class FeignLoadBalancer extends
	AbstractLoadBalancerAwareClient<FeignLoadBalancer.RibbonRequest, FeignLoadBalancer.RibbonResponse> {

	private final RibbonProperties ribbon;

	protected int connectTimeout;

	protected int readTimeout;

	protected IClientConfig clientConfig;

	protected ServerIntrospector serverIntrospector;

	/**
	* 注意此时的RibbonRequest参数的url属性已经从抽象的服务名经过ribbon负载均衡算法变成具体的域名或ip地址了
	* 负载均衡的逻辑在其父类AbstractLoadBalancerAwareClient.executeWithLoadBalancer()方法中
	*/
	@Override
	public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
		throws IOException {
		// http请求相关配置（connectTimeout、readTimeout）
		Request.Options options;
		if (configOverride != null) {
			RibbonProperties override = RibbonProperties.from(configOverride);
			options = new Request.Options(override.connectTimeout(this.connectTimeout),
										  override.readTimeout(this.readTimeout));
		}
		else {
			options = new Request.Options(this.connectTimeout, this.readTimeout);
		}
		
		// 关键代码： request.client()返回的是feign的Client接口的实现(OkHttpClient、ApacheHttpClient)
		// 它是在FeignClientFactoryBean.loadBalance()方法中将Client的实现设置到Feign.Builder中的
		// 这里调用http请求框架真正发送请求获取响应
		Response response = request.client().execute(request.toRequest(), options);
		// 将feign的Reponse对象包装为ribbon的response
		return new RibbonResponse(request.getUri(), response);
	}
}
```

- AbstractLoadBalancerAwareClient
```java
public abstract class AbstractLoadBalancerAwareClient<S extends ClientRequest, T extends IResponse> extends LoadBalancerContext implements IClient<S, T>, IClientConfigAware {

	public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
		LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

		try {
			// LoadBalancerCommand.submit()方法里面使用了rx-java，看起来很复杂。。。
			return command.submit(
				new ServerOperation<T>() {
					@Override
					public Observable<T> call(Server server) {
						// 这里负载均衡已经完成, server就是通过负载均衡算法选择的服务实例
						// reconstructURIWithServer就是把抽象地址转换为server的具体域名或ip
						// 得到最终http请求url
						URI finalUri = reconstructURIWithServer(server, request.getUri());
						// 将executeWithLoadBalancer方法参数的request中的uri替换成最终具体域名或ip的uri
						S requestForServer = (S) request.replaceUri(finalUri);
						try {
							// 调用了IClient接口中声明的 T execute(S request, IClientConfig requestConfig) 方法
							// 该方法由该类的子类FeignLoadBalancer实现了，来真正发出http请求
							return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
						} 
						catch (Exception e) {
							return Observable.error(e);
						}
					}
				})
				.toBlocking()
				.single();
		} catch (Exception e) {
			Throwable t = e.getCause();
			if (t instanceof ClientException) {
				throw (ClientException) t;
			} else {
				throw new ClientException(e);
			}
		}

	}
	
	protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
		RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);
		LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
			.withLoadBalancerContext(this)
			.withRetryHandler(handler)
			.withLoadBalancerURI(request.getUri());
		customizeLoadBalancerCommandBuilder(request, config, builder);
		return builder.build();
	}

}
```

### 3.3、**FeignLoadBalancerAutoConfiguration**

具有负载均衡功能的feign自动配置, 

注意
**@AutoConfigureAfter(FeignRibbonClientAutoConfiguration.class)**
这个自动化配置条件，表明这个类的自动化配置在```FeignRibbonClientAutoConfiguration```之后才执行，回顾上面的
```FeignRibbonClientAutoConfiguration```的执行流程可以知道，当启用ribbon时，```FeignRibbonClientAutoConfiguration```引入的3个类
```HttpClientFeignLoadBalancedConfiguration```、```OkHttpFeignLoadBalancedConfiguration```、```DefaultFeignLoadBalancedConfiguration```本身会创建Feign的```Client```接口的实现```LoadBalancerFeignClient```，所以由```FeignLoadBalancerAutoConfiguration```自动化配置的```FeignBlockingLoadBalancerClient```不会被创建，因为```Client```接口`的实现已经由```FeignRibbonClientAutoConfiguration```的自动化配置创建好了```LoadBalancerFeignClient```。

**下面的源码分析建立在spring.cloud.loadbalancer.ribbon.enabled=false的情况下进行分析的**

- FeignLoadBalancerAutoConfiguration源码如下：

```java
@ConditionalOnClass(Feign.class)
@ConditionalOnBean(BlockingLoadBalancerClient.class)
@AutoConfigureBefore(FeignAutoConfiguration.class)
@AutoConfigureAfter(FeignRibbonClientAutoConfiguration.class)
@EnableConfigurationProperties(FeignHttpClientProperties.class)
@Configuration(proxyBeanMethods = false)
// Order is important here, last should be the default, first should be optional
// see
// https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
@Import({ HttpClientFeignLoadBalancerConfiguration.class,
		OkHttpFeignLoadBalancerConfiguration.class,
		DefaultFeignLoadBalancerConfiguration.class })
class FeignLoadBalancerAutoConfiguration {

}
```
可以看到同时引入了带有负载均衡功能的apache httpclient、okhttp及默认实现的feign client实现

- DefaultFeignLoadBalancerConfiguration

Client.Default底层使用jdk自带的HttpURLConnection发送请求
```java
@Configuration(proxyBeanMethods = false)
class DefaultFeignLoadBalancerConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(BlockingLoadBalancerClient loadBalancerClient) {
		return new FeignBlockingLoadBalancerClient(new Client.Default(null, null),
				loadBalancerClient);
	}

}
```
- OkHttpFeignLoadBalancerConfiguration
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(OkHttpClient.class)
@ConditionalOnProperty("feign.okhttp.enabled")
@ConditionalOnBean(BlockingLoadBalancerClient.class)
@Import(OkHttpFeignConfiguration.class)
class OkHttpFeignLoadBalancerConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(okhttp3.OkHttpClient okHttpClient,
			BlockingLoadBalancerClient loadBalancerClient) {
		OkHttpClient delegate = new OkHttpClient(okHttpClient);
		return new FeignBlockingLoadBalancerClient(delegate, loadBalancerClient);
	}

}
```

可以看到无论是```DefaultFeignLoadBalancerConfiguration```还是```OkHttpFeignLoadBalancerConfiguration```都是使用```FeignBlockingLoadBalancerClient```，传入了具体http请求的实现，可以知道具体的负载均衡功能是由```FeignBlockingLoadBalancerClient```实现的

- FeignBlockingLoadBalancerClient
```java
class FeignBlockingLoadBalancerClient implements Client {
	private final Client delegate;

	private final BlockingLoadBalancerClient loadBalancerClient;

	@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		final URI originalUri = URI.create(request.url());
		String serviceId = originalUri.getHost();
		Assert.state(serviceId != null,
					 "Request URI does not contain a valid hostname: " + originalUri);
		// 使用负载均衡实现选择服务实例
		ServiceInstance instance = loadBalancerClient.choose(serviceId);
		if (instance == null) {
			// 无可用实例
			String message = "Load balancer does not contain an instance for the service "
				+ serviceId;
			if (LOG.isWarnEnabled()) {
				LOG.warn(message);
			}
			return Response.builder().request(request)
				.status(HttpStatus.SERVICE_UNAVAILABLE.value())
				.body(message, StandardCharsets.UTF_8).build();
		}
		
		// 将抽象的服务名替换成具体的域名、ip地址
		String reconstructedUrl = loadBalancerClient.reconstructURI(instance, originalUri)
			.toString();
		// 重新构造请求
		Request newRequest = Request.create(request.httpMethod(), reconstructedUrl,
											request.headers(), request.requestBody());
		// 委托给Client的具体实现（(OkHttpClient、ApacheHttpClient)）真正发出http请求
		return delegate.execute(newRequest, options);
	}
}
```

相比而言spring-cloud-loadbalancer与feign适配的代码比ribbon就简洁了很多很多。。。

# 4、总结

feign本身是一款优秀的开源组件，spring cloud feign又非常巧妙的将feign集成到spring boot中。
本文通过对spring cloud feign源代码的解读，详细的分析了feign集成到spring boot中的原理，使我们更加全面的了解到feign的使用。

spring cloud feign也是一个很好的学习spring boot的例子，从中我们可以学习到：

- spring boot注解声明注入bean
- spring类扫描机制
- spring接口动态注册bean
- spring命名空间隔离ApplicationContext

> 文章大部分来源于：http://techblog.ppdai.com/2018/05/28/20180528/  
> 本人在其基础上加入自己的理解