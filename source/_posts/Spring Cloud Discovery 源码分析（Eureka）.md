---
title: 3、Spring Cloud Discovery 源码分析（Eureka）
date: 2019-12-29 16:38:12
tags: spring cloud
categories: spring cloud
---


我们在将一个普通的Spring Boot应用注册到Eureka Server中，或是从Eureka Server中获取服务列表时，主要就做了两件事：

- 在应用主类he中配置了```@EnableDiscoveryClient```注解  
-  在`application.properties`中用`eureka.client.serviceUrl.defaultZone`参数指定了服务注册中心的位置  

顺着上面的线索，我们先查看具体实现:
- **@EnableDiscoveryClient的源码如下:**
```java
/**
 * Annotation to enable a DiscoveryClient implementation.
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 * @return - {@code true} if you want to automatically register.
	 */
	boolean autoRegister() default true;

}
```
`@EnableDiscoveryClient`注解的作用主要是用来引入`EnableDiscoveryClientImportSelector`这个类


- **EnableDiscoveryClientImportSelector的源码：**
```java
/**
 * @author Spencer Gibb
 */
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
		extends SpringFactoryImportSelector<EnableDiscoveryClient> {

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		String[] imports = super.selectImports(metadata);

		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
				metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

		boolean autoRegister = attributes.getBoolean("autoRegister");

		if (autoRegister) {
			List<String> importsList = new ArrayList<>(Arrays.asList(imports));
			importsList.add(
					"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
			imports = importsList.toArray(new String[0]);
		}
		else {
			Environment env = getEnvironment();
			if (ConfigurableEnvironment.class.isInstance(env)) {
				ConfigurableEnvironment configEnv = (ConfigurableEnvironment) env;
				LinkedHashMap<String, Object> map = new LinkedHashMap<>();
				map.put("spring.cloud.service-registry.auto-registration.enabled", false);
				MapPropertySource propertySource = new MapPropertySource(
						"springCloudDiscoveryClient", map);
				configEnv.getPropertySources().addLast(propertySource);
			}

		}

		return imports;
	}

	@Override
	protected boolean isEnabled() {
		return getEnvironment().getProperty("spring.cloud.discovery.enabled",
				Boolean.class, Boolean.TRUE);
	}

	@Override
	protected boolean hasDefaultFactory() {
		return true;
	}

}
```
`EnableDiscoveryClientImportSelector`继承了`SpringFactoryImportSelector`并指定了泛型`EnableDiscoveryClient`. 这里的泛型是重点.

- **SpringFactoryImportSelector源码**
```java
/**
 * Selects configurations to load, defined by the generic type T. Loads implementations
 * using {@link SpringFactoriesLoader}.
 *
 * @param <T> type of annotation class
 * @author Spencer Gibb
 * @author Dave Syer
 */
public abstract class SpringFactoryImportSelector<T>
		implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {
		
		private final Log log = LogFactory.getLog(SpringFactoryImportSelector.class);

	private ClassLoader beanClassLoader;

	private Class<T> annotationClass;

	private Environment environment;

	@SuppressWarnings("unchecked")
	protected SpringFactoryImportSelector() {
		this.annotationClass = (Class<T>) GenericTypeResolver
				.resolveTypeArgument(this.getClass(), SpringFactoryImportSelector.class);
	}
	
		@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		if (!isEnabled()) {
			return new String[0];
		}
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
				metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

		Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is "
				+ metadata.getClassName() + " annotated with @" + getSimpleName() + "?");

		// Find all possible auto configuration classes, filtering duplicates
		List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
				.loadFactoryNames(this.annotationClass, this.beanClassLoader)));

		if (factories.isEmpty() && !hasDefaultFactory()) {
			throw new IllegalStateException("Annotation @" + getSimpleName()
					+ " found, but there are no implementations. Did you forget to include a starter?");
		}

		if (factories.size() > 1) {
			// there should only ever be one DiscoveryClient, but there might be more than
			// one factory
			this.log.warn("More than one implementation " + "of @" + getSimpleName()
					+ " (now relying on @Conditionals to pick one): " + factories);
		}

		return factories.toArray(new String[factories.size()]);
	}
	
	protected abstract boolean isEnabled();
		
	...省略
}
```
这里只截取了部分变量和方法， `SpringFactoryImportSelector`是spring cloud common包中的一个抽象类, 主要作用是检查泛型T是否有指定的factory实现, 即`spring.factories`中有对应类的配置并启动自动化配置（由`SpringFactoriesLoader`加载并解析`spring.factories`文件, 具体加载原理见[springboot2.2自动注入文件spring.factories如何加载详解](https://my.oschina.net/zhaopeng2012/blog/3144983 "springboot2.2自动注入文件spring.factories如何加载详解")）

- **spring.factories**

在```spring-cloud-netflix-eureka-client.jar!/META-INF/spring.factories```中```EnableDiscoveryClient```的指定factory实现是
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.reactive.EurekaReactiveDiscoveryClientConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
```

`EnableAutoConfiguration`中包含了`EurekaClientAutoConfiguration`, `EurekaClientAutoConfiguration`会为```EurekaDiscoveryClientConfiguration`的实例依赖进行初始化。

下面对`spring.factories`中的eureka自动化配置一个个分析源码：
- **EurekaClientConfigServerAutoConfiguration 配置服务器自动化配置**
```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass({ EurekaInstanceConfigBean.class, EurekaClient.class,
		ConfigServerProperties.class })
public class EurekaClientConfigServerAutoConfiguration {

	@Autowired(required = false)
	private EurekaInstanceConfig instance;

	@Autowired(required = false)
	private ConfigServerProperties server;

	@PostConstruct
	public void init() {
		if (this.instance == null || this.server == null) {
			return;
		}
		String prefix = this.server.getPrefix();
		if (StringUtils.hasText(prefix) && !StringUtils
				.hasText(this.instance.getMetadataMap().get("configPath"))) {
			this.instance.getMetadataMap().put("configPath", prefix);
		}
	}

}
```

从源码看到注入了`EurekaInstanceConfig instance`配置，`EurekaInstanceConfig`这个bean是在`org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration#eurekaInstanceConfigBean()`中创建的， 另外 init()方法上有`@PostConstruct`注解，说明在创建这个bean后执行了init()方法， 方法内部获取`ConfigServerProperties` 中的prefix, 如果`eureka.instance.metadata.configPath`没有配置，则使用prefix的值。


- **EurekaClientAutoConfiguration 客户端自动化配置**

实例化application.yml中eureka.instance及eureka.client相关属性配置的bean以及创建EurekaClient

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@Import(DiscoveryClientOptionalArgsConfiguration.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
		CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {
		"org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
		"org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
		"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
	// spring cloud commons中的HasFeatures, 声明了使用了Eureka Client这个特性
	@Bean
	public HasFeatures eurekaFeature() {
		return HasFeatures.namedFeature("Eureka Client", EurekaClient.class);
	}
	
	/** 
	* 关键配置：eureka.client为前缀的相关配置属性bean(EurekaClientConfig的实现)
	*/
	@Bean
	@ConditionalOnMissingBean(value = EurekaClientConfig.class,
			search = SearchStrategy.CURRENT)
	public EurekaClientConfigBean eurekaClientConfigBean(ConfigurableEnvironment env) {
		EurekaClientConfigBean client = new EurekaClientConfigBean();
		if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
			// We don't register during bootstrap by default, but there will be another
			// chance later.
			client.setRegisterWithEureka(false);
		}
		return client;
	}
	
	/**
	* 关键配置：eureka.instance位前缀的配置属性bean（EurekaInstanceConfig的实现）
	*/
	@Bean
	@ConditionalOnMissingBean(value = EurekaInstanceConfig.class,
			search = SearchStrategy.CURRENT)
	public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils,
			ManagementMetadataProvider managementMetadataProvider) {
		// 获取配置的主机名
		String hostname = getProperty("eureka.instance.hostname");
		// 是否优先使用ip地址注册到注册中心
		boolean preferIpAddress = Boolean
				.parseBoolean(getProperty("eureka.instance.prefer-ip-address"));
		// 指定ip地址		
		String ipAddress = getProperty("eureka.instance.ip-address");
		// 是否启用安全端口 
		boolean isSecurePortEnabled = Boolean
				.parseBoolean(getProperty("eureka.instance.secure-port-enabled"));
		// servlet上下文路径
		String serverContextPath = env.getProperty("server.servlet.context-path", "/");
		// 启动端口号，如果没配置则默认8080
		int serverPort = Integer.parseInt(
				env.getProperty("server.port", env.getProperty("port", "8080")));
		// 管理端口号
		Integer managementPort = env.getProperty("management.server.port", Integer.class);
		// 管理上下文路径
		String managementContextPath = env
				.getProperty("management.server.servlet.context-path");
		// jmx端口号		
		Integer jmxPort = env.getProperty("com.sun.management.jmxremote.port",
				Integer.class);
		// 最终返回的bean		
		EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);

		instance.setNonSecurePort(serverPort);
		// 实例id， 默认为:主机名:服务名:[实例id | 端口号] 
		// 具体格式为${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id}或erver.port}
		instance.setInstanceId(getDefaultInstanceId(env));
		// 是否优先使用ip地址
		instance.setPreferIpAddress(preferIpAddress);
		instance.setSecurePortEnabled(isSecurePortEnabled);
		// ip地址
		if (StringUtils.hasText(ipAddress)) {
			instance.setIpAddress(ipAddress);
		}

		// 安全端口号是否启用
		if (isSecurePortEnabled) {
			instance.setSecurePort(serverPort);
		}
		// 主机名设置
		if (StringUtils.hasText(hostname)) {
			instance.setHostname(hostname);
		}
		// 实例状态页url路径
		String statusPageUrlPath = getProperty("eureka.instance.status-page-url-path");
		// 健康检查url路径
		String healthCheckUrlPath = getProperty("eureka.instance.health-check-url-path");

		if (StringUtils.hasText(statusPageUrlPath)) {
			instance.setStatusPageUrlPath(statusPageUrlPath);
		}
		if (StringUtils.hasText(healthCheckUrlPath)) {
			instance.setHealthCheckUrlPath(healthCheckUrlPath);
		}
		
		...省略代码

		setupJmxPort(instance, jmxPort);
		return instance;
	}
	
	/**
	* 关键配置：实例注册的实现bean（spring cloud commons项目中ServiceRegistry的实现）
	*/
	@Bean
	public EurekaServiceRegistry eurekaServiceRegistry() {
		return new EurekaServiceRegistry();
	}
	
	/**
	* 关键配置：启动时自动注册服务实现(spring cloud commons项目中AutoServiceRegistration的实现）)
	*/
	@Bean
	@ConditionalOnBean(AutoServiceRegistrationProperties.class)
	@ConditionalOnProperty(
			value = "spring.cloud.service-registry.auto-registration.enabled",
			matchIfMissing = true)
	public EurekaAutoServiceRegistration eurekaAutoServiceRegistration(
			ApplicationContext context, EurekaServiceRegistry registry,
			EurekaRegistration registration) {
		return new EurekaAutoServiceRegistration(context, registry, registration);
	}
	
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {
	
		@Autowired
		private ApplicationContext context;

		@Autowired
		private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

		/**
		* 关键配置：实例化netflix的EurekaClient， 提供最终的服务注册发现功能， 该类具体源码后续分析。
		*
		* 在该类的构造方法中会做非常多的事情：
		* 1、时候会根据eureka.client.serviceUrl的值依次遍历得到eureka server的地址向eureka server发送url路径
		*    为/apps的rest请求来获取已注册的服务信息，得到一个Applications对象，最终存储到localRegionApps属性中，
		*    如果获取失败则尝试使用当前zone的下一个url地址重新发送请求，直到成果(如果是集群，即eureka.client.serviceUrl的值是逗号分隔的多个地址)， 但是最多重试3次
		*
		* 2、 然后启动一系列的定时任务（cluster resolvers, heartbeat, instanceInfo replicator, fetch），具体源码
		*    后面会分析，这里只要知道什么时候第一次获取的服务注册信息、以后怎么更新的服务注册信息，存到哪就可以了。
		*/
		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
			return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
		}
		
		@Bean
		@ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
		public ApplicationInfoManager eurekaApplicationInfoManager(EurekaInstanceConfig config) {
			InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
			return new ApplicationInfoManager(config, instanceInfo);
		}
		
		/**
		* 关键配置：注册服务实例到注册中心时的承载实例信息的类（spring cloud commons项目中Registration的实现）
		*/
		@Bean
		@org.springframework.cloud.context.config.annotation.RefreshScope
		@ConditionalOnBean(AutoServiceRegistrationProperties.class)
		@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
		public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient, 
				CloudEurekaInstanceConfig instanceConfig,
				ApplicationInfoManager applicationInfoManager, 
				@Autowired(required = false) ObjectProvider<HealthCheckHandler> healthCheckHandler) {
			
			return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager)
					.with(eurekaClient).with(healthCheckHandler).build();
		}
	}
}
```


- **EurekaInstanceConfigBean 实例配置信息**  
实例化`eureka.instance`为前缀的配置信息`EurekaInstanceConfigBean`(实现了`EurekaInstanceConfig`)


用户承载eureka.instance配置信息的`EurekaInstanceConfigBean`类的源码如下：
```
import static org.springframework.cloud.commons.util.IdUtils.getDefaultInstanceId;

@ConfigurationProperties("eureka.instance")
public class EurekaInstanceConfigBean implements CloudEurekaInstanceConfig, EnvironmentAware {

	private static final String UNKNOWN = "unknown";
		
	// 当前机器信息
	private HostInfo hostInfo;
	
	// 应用名，setEnvironment(Enviroment)方法会设置该属性为spring.application.name的值
	private String appname = UNKNOWN;
	
	// 默认心跳时间， 每隔30秒客户端向服务端发送1次心跳，告诉服务端自己还活着，
	// 如果服务端在指定的leaseExpirationDurationInSeconds时间内还没有收到来自客户端的心跳，则服务端会从自己内部移除该实例, 这会导致禁止与该实例通信
	private int leaseRenewalIntervalInSeconds = 30;
	
	// 租约有效期，默认90秒，服务端从上一次心跳时间开始算起若经过90秒还没有收到来自客户端的心跳则会移除该实例
	// 该时间设置的过长会导致即使客户端已经死了，服务端仍然认为他还活着，那么当服务消费者真正访问该服务提供者实例时就会把请求路由给可已经死去的服务提供者，导致服务调用失败
	// 将此值设置得太小可能意味着，由于临时网络故障，实例可能会从流量中删除(比如10秒， 那么如果由于网络波动导致某次没能成功续约，但是客户端并不是真正死了，可能过一会就恢复了，服务端由于这1次网络波动没收到心跳就把客户端剔除了使得服务不可用就浪费了1个服务实例)
	private int leaseExpirationDurationInSeconds = 90;
	
	// 虚拟主机名，setEnvironment(Enviroment)方法会设置该属性为spring.application.name的值
	private String virtualHostName = UNKNOWN;
	
	// 实例id， 由EurekaClientAutoConfiguration设置默认值及最终值
	private String instanceId;
	
	// 实例元数据
	private Map<String, String> metadataMap = new HashMap<>();
	
	// 实例ip地址，由EurekaClientAutoConfiguration设置最终值，默认值由构造方法设置(若存在则设置）
	private String ipAddress;
	
	// 状态页url路径
	private String statusPageUrlPath = actuatorPrefix + "/info"
	
	// 健康检查页面地址
	private String healthCheckUrlPath = actuatorPrefix + "/health";
	
	// 命名空间
	private String namespace = "eureka";
	
	// 主机名，默认值由构造方法设置，最终值由EurekaClientAutoConfiguration设置（若存在则设置）
	private String hostname;
	
	//  是否优先使用ip地址注册，最终值由EurekaClientAutoConfiguration设置
	private boolean preferIpAddress = false;
	
	// 实例初始状态
	private InstanceStatus initialStatus = InstanceStatus.UP;
	
	// 实例环境变量， 当前类实现了EnvironmentAware接口，通过setEnvironment(Environment environment)回调方法赋值
	private Environment environment;
	
	public EurekaInstanceConfigBean(InetUtils inetUtils) {
		this.inetUtils = inetUtils;
		// 获取主机信息
		this.hostInfo = this.inetUtils.findFirstNonLoopbackHostInfo();
		// 设置默认的ip地址，由EurekaClientAutoConfiguration覆盖
		this.ipAddress = this.hostInfo.getIpAddress();
		//设置默认主机名，由EurekaClientAutoConfiguration覆盖
		this.hostname = this.hostInfo.getHostname();
	}
	
	// 当前类实现了EnvironmentAware接口
	@Override
	public void setEnvironment(Environment environment) {
		this.environment = environment;

		//取spring.application.name的值
		String springAppName = this.environment.getProperty("spring.application.name", "");
		if (StringUtils.hasText(springAppName)) {
			setAppname(springAppName);
			setVirtualHostName(springAppName);
			setSecureVirtualHostName(springAppName);
		}
	}
}

```
从EurekaInstanceConfigBean的构造方法中可以看到它接收1个参数`InetUtils`,  `this.hostInfo = this.inetUtils.findFirstNonLoopbackHostInfo();`这行代码使用传入的InetUtils获取主机信息， 接下来分析InetUtils网络工具类源码。
- **InetUtils网络工具类**
```
public class InetUtils implements Closeable {
	
	public HostInfo findFirstNonLoopbackHostInfo() {
		InetAddress address = findFirstNonLoopbackAddress();
		if (address != null) {
			return convertAddress(address);
		}
		HostInfo hostInfo = new HostInfo();
		hostInfo.setHostname(this.properties.getDefaultHostname());
		hostInfo.setIpAddress(this.properties.getDefaultIpAddress());
		return hostInfo;
	}
	
	public InetAddress findFirstNonLoopbackAddress() {
		InetAddress result = null;
		try {
			int lowest = Integer.MAX_VALUE;
			// 通过jdk的NetworkInterface获取所有网络接口
			for (Enumeration<NetworkInterface> nics = NetworkInterface
					.getNetworkInterfaces(); nics.hasMoreElements();) {
				NetworkInterface ifc = nics.nextElement();
				// 当前遍历的网络接口是否已启用
				if (ifc.isUp()) {
					this.log.trace("Testing interface: " + ifc.getDisplayName());
					// 找出索引最小的网络接口
					if (ifc.getIndex() < lowest || result == null) {
						lowest = ifc.getIndex();
					}
					else if (result != null) {
						continue;
					}

					// 判断是否需要忽略当前网络接口
					if (!ignoreInterface(ifc.getDisplayName())) {
						// 不忽略当前网络接口，获取当前网络接口的所有网络地址
						for (Enumeration<InetAddress> addrs = ifc
								.getInetAddresses(); addrs.hasMoreElements();) {
							InetAddress address = addrs.nextElement();
							// 是ipv4地址，且不是本地回环地址(127.xxx.xxx.xx这种地址)， 且是优先需要使用的地址
							if (address instanceof Inet4Address
									&& !address.isLoopbackAddress()
									&& isPreferredAddress(address)) {
								this.log.trace("Found non-loopback interface: "
										+ ifc.getDisplayName());
								result = address;
							}
						}
					}
					// @formatter:on
				}
			}
		}
		catch (IOException ex) {
			this.log.error("Cannot get first non-loopback address", ex);
		}

		if (result != null) {
			return result;
		}

		try {
			return InetAddress.getLocalHost();
		}
		catch (UnknownHostException e) {
			this.log.warn("Unable to retrieve localhost");
		}

		return null;
	}
}
```

`org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration#eurekaInstanceConfigBean`方法中获取`EurekaInstanceConfigBean`实例时设置了实例id(`instance.setInstanceId(getDefaultInstanceId(env));`)中的`getDefaultInstanceId()`方法是`IdUtils`类中的方法，这里的`getDefaultInstanceId()`方法是静态导入的，所以没有看到通过Class.methods()这种形式调用，`IdUtils`的实现源码如下：

- **IdUtils 实例id工具类**
```java
public final class IdUtils {
	public static String getDefaultInstanceId(PropertyResolver resolver) {
		return getDefaultInstanceId(resolver, true);
	}

	// 默认实例id(主机名:服务名:[实例id | 端口号])
	// 具体格式为${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id}或${server.port}
	public static String getDefaultInstanceId(PropertyResolver resolver,
			boolean includeHostname) {
		String vcapInstanceId = resolver.getProperty("vcap.application.instance_id");
		if (StringUtils.hasText(vcapInstanceId)) {
			return vcapInstanceId;
		}

		String hostname = null
		// 实例id是否包含主机名，上面重载的getDefaultInstanceId方法内部传递的第2个参数为true，所以会包含主机名	;
		if (includeHostname) {
			hostname = resolver.getProperty("spring.cloud.client.hostname");
		}
		// spring.application.name的属性值
		String appName = resolver.getProperty("spring.application.name");
		//  值为${spring.cloud.client.hostname}:${spring.application.name} , 把hostname和appName用冒号拼接起来
		String namePart = combineParts(hostname, SEPARATOR, appName);
		// 值为${spring.application.instance_id}或${server.port}
		String indexPart = resolver.getProperty("spring.application.instance_id",
				resolver.getProperty("server.port"));

		//  把namePart和indexPart用冒号拼接起来
		// ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id}或${server.port}
		return combineParts(namePart, SEPARATOR, indexPart);
	}

	public static String combineParts(String firstPart, String separator,
			String secondPart) {
		String combined = null;
		if (firstPart != null && secondPart != null) {
			combined = firstPart + separator + secondPart;
		}
		else if (firstPart != null) {
			combined = firstPart;
		}
		else if (secondPart != null) {
			combined = secondPart;
		}
		return combined;
	}
}	
```
- **EurekaClientConfigBean 客户端配置**

实例化eureka.client为前缀的配置信息EurekaClientConfigBean(实现了EurekaClientConfig)

```java
@ConfigurationProperties(EurekaClientConfigBean.PREFIX)
public class EurekaClientConfigBean implements EurekaClientConfig, Ordered {
	
	private static final int MINUTES = 60;
	
	// 配置前缀
	public static final String PREFIX = "eureka.client";

	// 默认eureka server的地址
	public static final String DEFAULT_URL = "http://localhost:8761" + DEFAULT_PREFIX + "/";
	
	// 表示默认zone的名称
	// region可以理解为数据中心, zone可以理解为1个数据中的机房， instance理解为具体主机
	// 例如 亚马逊、阿里云2个数据中心
	// 阿里云有多个跨区的机房，如北京、广东
	// 而1个机房里又有很多主机（实例）
	public static final String DEFAULT_ZONE = "defaultZone";
	
	// 每隔多长时间从eureka server拉取1次注服务册信息，单位秒
	private int registryFetchIntervalSeconds = 30;
	
	// 指示将实例更改信息复制到eureka服务器的频率(以秒为单位)
	private int instanceInfoReplicationIntervalSeconds = 30;
	
	// 指示最初(以秒为单位)将实例信息复制到eureka需要多长时间
	private int initialInstanceInfoReplicationIntervalSeconds = 40;
	
	// 指定多久1次轮训eureka server服务器信息的变更， Eureka服务器可以被添加或移除，该设置可以控制Eureka客户端应该多快知道eureka server的添加和移除。
	private int eurekaServiceUrlPollIntervalSeconds = 5 * MINUTES;
	
	// 代理端口
	private String proxyPort;
	// 代理主机
	private String proxyHost;
	// 代理用户名
	private String proxyUserName;
	// 代理密码
	private String proxyPassword;
	
	// 从eureka server读信息的超时时间设置
	private int eurekaServerReadTimeoutSeconds = 8;
	
	// 与eureka server建立连接的超时时间设置
	private int eurekaServerConnectTimeoutSeconds = 5;
	
	// 从eureka server获取注册信息的后备实现（须实现BackupRegistry接口）
	private String backupRegistryImpl;
	/**
	* 指示eureka客户端是否应该使用DNS机制获取要与之通信的eureka服务器列表。
	* 当更新DNS名称以拥有其他服务器时，将在eureka客户端轮询eurekaServiceUrlPollIntervalSeconds中指定的信息之后立即使用该信息。
	*/
	private boolean useDnsForFetchingServiceUrls = false;
	
	/**
	* 获取用来构造服务url的上下文路径，以便在eureka的服务器地址列表来源于dns时与eureka进行通信，当从eurekaServerServiceUrls返回服务url信息时该配置是不需要的。
	*
	* 当useDnsForFetchingServiceUrls设置为true时，将使用DNS机制，eureka客户端希望DNS以某种方式配置，以便它能够动态获取变化的eureka服务器。
	*/
	private String eurekaServerURLContext;
	
	/**
	* 获取用来构造服务url的端口，以便在eureka的服务器地址列表来源于dns时与eureka进行通信，当从eurekaServerServiceUrls返回服务url信息时该配置是不需要的。
	*
	* 当useDnsForFetchingServiceUrls设置为true时，将使用DNS机制，eureka客户端希望DNS以某种方式配置，以便它能够动态获取变化的eureka服务器。
	*/
	private String eurekaServerPort;
	
	/**
	* 获取要查询的DNS名称，以获得eureka服务器的列表。如果契约通过实现serviceUrls返回服务url，则不需要此信息。
	*
	* 当useDnsForFetchingServiceUrls设置为true时，将使用DNS机制，eureka客户端希望DNS以某种方式配置，以便它能够动态获取变化的eureka服务器。
	* 更改在运行时有效。
	*
	private String eurekaServerDNSName;
	
	// 数据中心名称
	private String region = "us-east-1";
	
	// 指定与eureka server建立的http连接在关闭前可以空闲多少时间（以秒位单位）
	private int eurekaConnectionIdleTimeoutSeconds = 30;
	
	// 心跳线程池的初始大小
	private int heartbeatExecutorThreadPoolSize = 2
	// 心跳执行器指数回退相关属性。它是重试延迟的最大乘法器值，用于发生一系列超时的情况。	
	private int heartbeatExecutorExponentialBackOffBound = 10;
	
	// cacheRefreshExecutor线程池的初始大小。
	private int cacheRefreshExecutorThreadPoolSize = 2;
	//  cacheRefreshExecutor执行器指数回退相关属性。它是重试延迟的最大乘法器值，用于发生一系列超时的情况。	
	private int cacheRefreshExecutorExponentialBackOffBound = 10;
	
	// eureka服务地址，这里设置了默认值defaultZone=http://localhost:8761/eureka/
	private Map<String, String> serviceUrl = new HashMap<>();
	{
		this.serviceUrl.put(DEFAULT_ZONE, DEFAULT_URL);
	}
	
	// 指定当前实例是否应该将它自己的信息注册到eureka server中用于服务发现
	private boolean registerWithEureka = true;
	
	// 指定当前实例是否应出于延迟和/或其他原因尝试使用同一区域中的eureka服务器。
	// 理想情况下，eureka客户端被配置为与同一区域中的服务器通信
	// 正如 registryFetchIntervalSeconds 指定的那样，这些更改在运行时在下一个注册表获取周期生效
	private boolean preferSameZoneEureka = true;
	
	// 用逗号分隔的区域(region)列表，用于获取eureka注册信息， 必须为每个region定义可用机房（availability zones）来作为availabilityZones的返回值，如果不这样做会导致启动失败
	private String fetchRemoteRegionsRegistry;
	
	// 获取此实例所在区域的可用性区域列表(在AWS数据中心中使用)。
	// 正如 registryFetchIntervalSeconds 指定的那样，这些更改在运行时在下一个注册表获取周期生效
	private Map<String, String> availabilityZones = new HashMap<>();
	
	// 指定是否在筛选只具有 instanceatus UP 状态的实例的应用程序后获取应用程序。
	private boolean filterOnlyUpInstances = true;
	
	// 指定当前客户端是否应该从eureka server获取服务注册信息
	private boolean fetchRegistry = true;
	
	// 指定服务端是否可以将一个客户端请求重定向到后备服务器或者集群中
	private boolean allowRedirects = false;
	
	// 如果设置为true，通过ApplicationInfoManager进行的本地状态更新将触发按需(但速率有限)注册/更新到远程eureka服务器。
	private boolean onDemandUpdateStatusChange = true;
	
	/**
	* 根据region获取机房列表， 如果找不到则返回默认机房(defaultZone)
	*/
	@Override
	public String[] getAvailabilityZones(String region) {
		// 根据region获取机房列表
		String value = this.availabilityZones.get(region);
		if (value == null) {
			// 默认机房
			value = DEFAULT_ZONE;
		}
		return value.split(",");
	}
	
	/**
	* 根据机房获取eureka server地址， 一个机房可以部署多台eureka server组成高可用集群
	*/
	@Override
	public List<String> getEurekaServerServiceUrls(String myZone) {
		// 根据机房id获取eureka server地址（逗号分隔的多个url）
		String serviceUrls = this.serviceUrl.get(myZone);
		if (serviceUrls == null || serviceUrls.isEmpty()) {
			// 使用默认机房(defaultZone)的eureka server地址
			serviceUrls = this.serviceUrl.get(DEFAULT_ZONE);
		}
		if (!StringUtils.isEmpty(serviceUrls)) {
			// 用逗号分隔
			final String[] serviceUrlsSplit = StringUtils
					.commaDelimitedListToStringArray(serviceUrls);
			List<String> eurekaServiceUrls = new ArrayList<>(serviceUrlsSplit.length);
			for (String eurekaServiceUrl : serviceUrlsSplit) {
				// eureka server地址如果不是以/结尾则追加/
				if (!endsWithSlash(eurekaServiceUrl)) {
					eurekaServiceUrl += "/";
				}
				eurekaServiceUrls.add(eurekaServiceUrl.trim());
			}
			return eurekaServiceUrls;
		}

		return new ArrayList<>();
	}
}	
```
# Eureka 中的 region 和 Zone
## 背景
像亚马逊这种大型的跨境电商平台，会有很多个机房。这时如果上线一个服务的话，我们希望一个机房内的服务优先调用同一个机房内的服务，当同一个机房的服务不可用的时候，再去调用其它机房的服务，以达到减少延时的作用。
于是亚马逊的 AWS 提供了 region 和 zone 两个概念

## 概念
- region：可以简单理解为地理上的分区。比如亚洲地区，或者华北地区，再或者北京地区等等，没有具体大小的限制，根据项目具体的情况，可以自行划分region。
- zone：可以简单理解为 region 内的具体机房，比如说 region 划分为华北地区，然后华北地区有两个机房，就可以在此 region 之下划分出 zone1、zone2 两个 zone

eureka 也借用了 region 和 zone 的概念

##  分区服务架构图  
![](https://oscimg.oschina.net/oscnet/up-ead4f9c1eb120c96fd03904c821accd0018.png)

如图所示，有一个 region:华北地区，下面有两个机房，机房A 和机房B
每个机房内有一个 Eureka Server 集群 和两个服务提供者 ServiceA 和 ServerB
现在假设 serverA 需要调用 ServerB 服务，按照就近原则，serverA 会优先调用同一个 zone 内的 ServiceB，当 ServiceB 不可用时，才会去调用另一个 zone 内的 ServiceB

## Eureka 中 Regin 和 Zone 的相关配置
- 服务注册：要保证服务注册到同一个zone内的注册中心，因为如果注册到别zone的注册中心的话，网络延时比较大，心跳检测很可能出问题。
- 服务调用：要保证优先调用同一个zone内的服务，只有在同一个zone内的服务不可用时，才去调用别zone的服务。
```yml
eureka:
  client:
    # 尽量向同一区域的 eureka 注册,默认为true
    prefer-same-zone-eureka: true
    #地区
    region: huabei
    availability-zones:
      huabei: zone-1,zone-2
    service-url:
      zone-1: http://localhost:30000/eureka/
      zone-2: http://localhost:30001/eureka/

```
当存在多个注册中心时，选择逻辑为:
1. 如果 prefer-same-zone-eureka 为 false，按照 service-url 下的 list 取第一个注册中心来注册，并和其维持心跳检测，不再向list内的其它的注册中心注册和维持心跳。
只有在第一个注册失败的情况下，才会依次向其它的注册中心注册，总共重试3次，如果3个service-url都没有注册成功，则注册失败。  
注册失败后每隔一个心跳时间，会再次尝试。


2. 如果 prefer-same-zone-eureka 为true，先通过 region 取 availability-zones 内的第一个zone，然后通过这个zone取 service-url 下的list，并向list内的第一个注册中心进行注册和维持心跳，不再向list内的其它的注册中心注册和维持心跳。
只有在第一个注册失败的情况下，才会依次向其它的注册中心注册，总共重试3次，如果3个service-url都没有注册成功，则注册失败。  
注册失败后每隔一个心跳时间，会再次尝试。

为了保证服务注册到同一个 zone 的注册中心，一定要注意 availability-zones 的顺序，必须把同一 zone 写在最前面

## 服务调用
```yml
eureka:
  instance:
    # 服务和注册中心的心跳间隔时间，默认为30s
    lease-renewal-interval-in-seconds: 30
    # 服务和注册中心的心跳超时时间，默认为90s
    lease-expiration-duration-in-seconds: 90
    metadata-map:
      # 当前服务所属的 zone
      zone: zone1

```
服务消费者和服务提供者分别属于哪个zone，均是通过 eureka.instance.metadata-map.zone 来判定的。  

服务消费者会先通过 ribbon 去注册中心拉取一份服务提供者的列表，然后通过 eureka.instance.metadata-map.zone 指定的 zone 进行过滤，过滤之后如果同一个 zone 内的服务提供者有多个实例，则会轮流调用。  

只有在同一个 zone 内的所有服务提供者都不可用时，才会调用其它zone内的服务提供者。


>作者：我妻礼弥  
>链接：https://juejin.im/post/5d68b73af265da03b12061be  
>来源：掘金  
>著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


- **EurekaClient**  
以下图片来自Netflix官方，图中显示Eureka Client会向注册中心发起Get Registry请求来获取服务列表：
![](https://oscimg.oschina.net/oscnet/up-3ba973845dacb74c2edf4b188caf17e14d1.png)
在`org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration`中实例化了该bean, 源码如下：
```
public class EurekaClientAutoConfiguration{

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {
	
		@Autowired
		private ApplicationContext context;

		@Autowired
		private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

		/**
		* 关键配置：实例化netflix的EurekaClient， 提供最终的服务注册发现功能， 该类具体源码后续分析。
		*
		* 在该类的构造方法中会做非常多的事情：
		* 1、时候会根据eureka.client.serviceUrl的值依次遍历得到eureka server的地址向eureka server发送url路径
		*    为/apps的rest请求来获取已注册的服务信息，得到一个Applications对象，最终存储到localRegionApps属性中，
		*    如果获取失败则尝试使用当前zone的下一个url地址重新发送请求，直到成果(如果是集群，即eureka.client.serviceUrl的值是逗号分隔的多个地址)， 但是最多重试3次
		*
		* 2、 然后启动一系列的定时任务（cluster resolvers, heartbeat, instanceInfo replicator, fetch），具体源码
		*    后面会分析，这里只要知道什么时候第一次获取的服务注册信息、以后怎么更新的服务注册信息，存到哪就可以了。
		*/
		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
			return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
		}
	}
```

下面看`EurekaClient`这个类的源码，注意重点是该类的**构造方法创建了一系列的定时任务（心跳、更新服务注册信息、根据dns更新serviceUrl、注册实例信息到eurekaserver）以及注册了事件监听器StatusChangeListener用于监听实例自身状态变化，当发生变化时上报服务实例信息到eureka server**

该类的继承关系如下：  
![](https://oscimg.oschina.net/oscnet/up-b975a3a81fb408329867b2d1795f4061061.png)

```java
@Singleton
public class DiscoveryClient implements EurekaClient {
	// 用于执行如下几种列定时任务：更新服务地址、
	private final ScheduledExecutorService scheduler;
	// additional executors for supervised subtasks
	private final ThreadPoolExecutor heartbeatExecutor;
	private final ThreadPoolExecutor cacheRefreshExecutor;
	
	// 存储调用/apps这个Url从eureka server获取的服务注册信息， 返回的信息被封装成Applications对象
	// 而Applications对象内部会持有private final AbstractQueue<Application> applications;这个队列，1个Application代表1个服务，1个服务可能有多个实例，
	private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();
	
	...省略部分代码
	
	@Inject
 	DiscoveryClient(ApplicationInfoManager applicationInfoManager, 
					EurekaClientConfig config, 
					AbstractDiscoveryClientOptionalArgs args,
					Provider<BackupRegistry> backupRegistryProvider, 
					EndpointRandomizer endpointRandomizer) {
		this.applicationInfoManager = applicationInfoManager;
 		InstanceInfo myInfo = applicationInfoManager.getInfo();

		clientConfig = config;
		staticClientConfig = clientConfig;
		transportConfig = config.getTransportConfig();
		instanceInfo = myInfo;
		
		 ...省略部分代码
		
		 try {
			 // default size of 2 - 1 each for heartbeat and cacheRefresh
			 scheduler = Executors.newScheduledThreadPool(2,
					new ThreadFactoryBuilder()
						.setNameFormat("DiscoveryClient-%d")
						.setDaemon(true)
						 .build());

			 heartbeatExecutor = new ThreadPoolExecutor(
				 1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
				 new SynchronousQueue<Runnable>(),
				 new ThreadFactoryBuilder()
				 .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
				 .setDaemon(true)
				 .build()
			 );  // use direct handoff

			 cacheRefreshExecutor = new ThreadPoolExecutor(
				 1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
				 new SynchronousQueue<Runnable>(),
				 new ThreadFactoryBuilder()
				 .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
				 .setDaemon(true)
				 .build()
			 );  // use direct handoff

			 eurekaTransport = new EurekaTransport();
			 scheduleServerEndpointTask(eurekaTransport, args);

			 AzToRegionMapper azToRegionMapper;
			 if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
				 azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
			 } else {
				 azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
			 }
			 if (null != remoteRegionsToFetch.get()) {
				 azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
			 }
			 instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
		 } catch (Throwable e) {
			 throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
		 }
		
		// 关键代码：第1次发送/apps这个rest请求从eureka server获取所有的服务注册信息，并封装到Applications对象中，然后存储到localRegionApps这个属性中
		if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
			fetchRegistryFromBackup();
		}

		// 在所有的后台 任务启动前调用并执行前置注册处理器
		if (this.preRegistrationHandler != null) {
			this.preRegistrationHandler.beforeRegistration();
		}
		
		// 是否应该将自己注册到eureka server中，并且强制在初始化的时候注册，默认false，不会进入if中
		if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
			try {
				if (!register() ) {
					throw new IllegalStateException("Registration error at startup. Invalid server response.");
				}
			} catch (Throwable th) {
				logger.error("Registration error at startup: {}", th.getMessage());
				throw new IllegalStateException(th);
			}
		}
		
		// finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
		// 关键代码：初始化定时任务（比如：集群解析、心跳、实例信息复制、更新服务注册信息）
		initScheduledTasks();
		
		// 将自己放到单例对象DiscoveryManager中，这样可以在任意位置很方便的获取EurekaClient的相关信息
		DiscoveryManager.getInstance().setDiscoveryClient(this);
		// 存储客户端配置信息到DiscoveryManager中
		DiscoveryManager.getInstance().setEurekaClientConfig(config);
	}
	
	/**
	* 服务注册实现
	*/
	boolean register() throws Throwable {
		EurekaHttpResponse<Void> httpResponse;
		try {
			httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
		} catch (Exception e) {
			logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
			throw e;
		}
	}
	
	/**
	* 发送/apps请求到eureka server获取所有服务注册信息，并存储到localRegionApps属性中
	* 在第一次调用时会获取全部的服务注册信息，以后调用该方法只会获取增量的服务注册信息
	*/
	 private boolean fetchRegistry(boolean forceFullRegistryFetch) {
		 
		 Applications applications = getApplications();

		 
		 if (clientConfig.shouldDisableDelta() // 是否禁用增量获取
			 || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
			 || forceFullRegistryFetch // 是否强制刷新，获取全量的服务注册信息
			 || (applications == null) // 是否已经获取过服务注册信息
			 || (applications.getRegisteredApplications().size() == 0)
			 || (applications.getVersion() == -1)) { // 没有获取过
			
			 // 获取全量的服务注册信息
			getAndStoreFullRegistry();	
		}
		else {
			//获取增量的服务注册信息
			 getAndUpdateDelta(applications);
		}
		 
		...省略部分代码
	 }
	
	/**
	* 服务发现：
	*
	* 发送/apps请求到eureka server全量获取所有服务注册信息，并存储到localRegionApps属性中
	*/
	private void getAndStoreFullRegistry() throws Throwable {
		Applications apps = null;
		EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
			// 真正发送/apps请求获取服务注册信息
			? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get()) 
			: eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
		if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
			apps = httpResponse.getEntity();
		}
		
		if (apps == null) {
			logger.error("The application is null for some reason. Not storing this information");
		} 
		else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
			// 1、 filterAndShuffle方法返回的Applications对象中除了维护全部的服务注册信息集合外，
			//     这里还通过过滤操作维护了一个服务状态正常的服务注册信息集合
			// 2、将Applications对象存储到localRegionApps属性中
			localRegionApps.set(this.filterAndShuffle(apps));
			
			logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
		}
		else {
			logger.warn("Not updating applications as another thread is updating it already");
		}
	}
	
	/**
	* 返回localRegionApps属性中保存的服务注册信息
	*/
	@Override
	public Applications getApplications() {
		return localRegionApps.get();
	}
	
	
	/**
	* 初始化所有的定时任务
	*/
	private void initScheduledTasks() {
		// 是否需要从eureka server获取服务注册信息
		if (clientConfig.shouldFetchRegistry()) {
			// registry cache refresh timer
			int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
			int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
			// 启动定时任务， 获取服务注册信息
			scheduler.schedule(
				new TimedSupervisorTask(
					"cacheRefresh",
					scheduler,
					cacheRefreshExecutor,
					registryFetchIntervalSeconds,
					TimeUnit.SECONDS,
					expBackOffBound,
					// 获取服务注册信息的任务
					new CacheRefreshThread()
				),
				registryFetchIntervalSeconds, TimeUnit.SECONDS);
		}

		// 是否应该将自己注册到eureka server上，如果需要则启动心跳线程向服务端发送/renew请求进行续约
		if (clientConfig.shouldRegisterWithEureka()) {
			int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
			int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
			logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

			// 启动心跳定时任务
			scheduler.schedule(
				new TimedSupervisorTask(
					"heartbeat",
					scheduler,
					heartbeatExecutor,
					renewalIntervalInSecs,
					TimeUnit.SECONDS,
					expBackOffBound,
					// 发送/renew请求的任务
					new HeartbeatThread()
				),
				renewalIntervalInSecs, TimeUnit.SECONDS);

			// 上报自身信息到eureka server的定时任务， 它实现了Runnable接口
			instanceInfoReplicator = new InstanceInfoReplicator(
				this,
				instanceInfo,
				clientConfig.getInstanceInfoReplicationIntervalSeconds(),
				2); // burstSize

			// 监听本地实例状态变更（如由UP变为DOWN状态）
			statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
				@Override
				public String getId() {
					return "statusChangeListener";
				}

				@Override
				public void notify(StatusChangeEvent statusChangeEvent) {
					if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
						InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
						// log at warn level if DOWN was involved
						logger.warn("Saw local status change event {}", statusChangeEvent);
					} else {
						logger.info("Saw local status change event {}", statusChangeEvent);
					}
					
					// 实例自身状态发生变更，立即注册实例信息到eureka server
					instanceInfoReplicator.onDemandUpdate();
				}
			};

			if (clientConfig.shouldOnDemandUpdateStatusChange()) {
				// 注册实例状态变更监听器， 在com.netflix.appinfo.ApplicationInfoManager#setInstanceStatus中发布了该事件
				applicationInfoManager.registerStatusChangeListener(statusChangeListener);
			}

			// 内部会使用Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);启动当前定时任务
			instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
		}
		else {
			logger.info("Not registering with Eureka server per configuration");
		}
	}
	
	
	
	@VisibleForTesting
 	void refreshRegistry() {
		
		try {
			...省略部分代码
				
			// 重新从eureka server获取服务注册列表	
			boolean success = fetchRegistry(remoteRegionsModified);
			if (success) {
				registrySize = localRegionApps.get().size();
				lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
			}
		} catch (Throwable e) {
			logger.error("Cannot fetch registry from server", e);
		}
	｝
	
	/**
	* 用于服务注册定时任务，见com.netflix.discovery.InstanceInfoReplicator#run
	* 该方法会重新获取本地配置信息变化，如果变化了会调用com.netflix.appinfo.InstanceInfo#setIsDirty()
	* 将实例状态设置已变更脏数据状态，以便InstanceInfoReplicator任务感知到实例状态变化将心的实例信息注册到eureka server
	*/
	void refreshInstanceInfo() {
		// 刷新hostname、ipAddress变更信息
		applicationInfoManager.refreshDataCenterInfoIfRequired();
		
		// 刷新租约配置变更（leaseExpirationDurationInSeconds、leaseRenewalIntervalInSeconds）
		applicationInfoManager.refreshLeaseInfoIfRequired();
		
		...省略部分代码
	｝
	
	/**
	* 从eureka server刷新服务注册信息的线程
	*/
	class CacheRefreshThread implements Runnable {
		public void run() {
			refreshRegistry();
		}
	}	
		
	/**
  	 * 心跳线程任务
  	 */
	private class HeartbeatThread implements Runnable {

		public void run() {
			if (renew()) {
				lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
			}
		}
	}
}
```

- **InstanceInfoReplicator**
1. InstanceInfoReplicator是个任务类，负责将自身的信息周期性的上报到Eureka server；
2. 有两个场景触发上报：周期性任务、服务状态变化(onDemandUpdate被调用)，因此，在同一时刻有可能有两个上报的任务同时出现；
3. 单线程执行上报的操作，如果有多个上报任务，也能确保是串行的；
4. 有频率限制，通过burstSize参数来控制；
5. 先创建的任务总是先执行，但是onDemandUpdate方法中创建的任务会将周期性任务给丢弃掉；
```java
/**
* 用于更新本地instanceinfo并将其复制到远程服务器的任务。这个任务的属性是:
* 1. 使用单个更新线程进行配置，以确保对远程服务器进行连续更新
* 2. 可以通过onDemandUpdate()按需调度更新任务
* 3. 任务处理的速率受到burstSize的限制
* 4. 新的更新任务总是在较早的更新任务之后自动调度。但是，如果启动了随需应变任务，
*    则会丢弃调度的自动更新任务(并在* 新的随需应变更新之后调度新的自动更新任务)。
*/
class InstanceInfoReplicator implements Runnable {
	private final DiscoveryClient discoveryClient;
	private final InstanceInfo instanceInfo;

	private final int replicationIntervalSeconds;
	private final ScheduledExecutorService scheduler;
	
	private final AtomicReference<Future> scheduledPeriodicRef;

	private final AtomicBoolean started;
	private final RateLimiter rateLimiter;
	private final int burstSize;
	private final int allowedRatePerMinute;
	
	InstanceInfoReplicator(DiscoveryClient discoveryClient,
		InstanceInfo instanceInfo,
		int replicationIntervalSeconds, 
		int burstSize) {
		
		this.discoveryClient = discoveryClient;
		this.instanceInfo = instanceInfo;
		//线程池，core size为1，使用DelayedWorkQueue队列
		this.scheduler = Executors.newScheduledThreadPool(1,
				new ThreadFactoryBuilder()
					.setNameFormat("DiscoveryClient-InstanceInfoReplicator-%d")
					.setDaemon(true)
					.build());

		this.scheduledPeriodicRef = new AtomicReference<Future>();

		this.started = new AtomicBoolean(false);
		this.rateLimiter = new RateLimiter(TimeUnit.MINUTES);
		this.replicationIntervalSeconds = replicationIntervalSeconds;
		this.burstSize = burstSize;
 		//通过周期间隔，和burstSize参数，计算每分钟允许的任务数
		this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
	}
	
	/**
	* 启动定时任务（通过scheduledPeriodicRef持有的引用可以获得启动的任务，并可以取消该定时任务）
	*/
	public void start(int initialDelayMs) {
		if (started.compareAndSet(false, true)) { // cas更新设置为为已启用状态
			/**
			* setIsDirty()方法的作用是：设置脏标志，以便在下一次心跳时将实例信息传送到发现服务器，com.netflix.appinfo.InstanceInfo#setIsDirty()内部代码为：
			*
			* isInstanceInfoDirty = true;
  			* lastDirtyTimestamp = System.currentTimeMillis();
			*/
			
			// 这里是为了启动后立即将实例信息上报到eureka server
			instanceInfo.setIsDirty(); 
			// 启动定时任务， 执行run方法中的逻辑
			Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);
			// 持有启动的任务，后续可以获得该任务然后调用其cancel方法取消执行
			scheduledPeriodicRef.set(next);
		}
	}
	
	/**
	* com.netflix.discovery.DiscoveryClient#initScheduledTasks类中的statusChangeListener实例状态事件监听器中的notify放啊会调用该方法，当实例状态发生变化时立即同步（取消定时未完成的任务）实例信息到eureka server
	*/
	public boolean onDemandUpdate() {
		 // 没有达到频率限制才会执行
		if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
			if (!scheduler.isShutdown()) {
				//提交一个任务
				scheduler.submit(new Runnable() {
					@Override
					public void run() {
						logger.debug("Executing on-demand update of local InstanceInfo");

						// 获取正在执行的定时任务
						Future latestPeriodic = scheduledPeriodicRef.get();
						// 如果当前定时任务启动了，但是还没有执行完成，则立即取消任务
						if (latestPeriodic != null && !latestPeriodic.isDone()) {
							logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
							// 取消本次正在执行的定时任务（仅仅是取消本次任务，下一个任务周期到了仍然会继续执行）
							latestPeriodic.cancel(false);
						}

						// 直接调用run方法将变更的实例信息注册到eureka server
						InstanceInfoReplicator.this.run();
					}
				});
				return true;
			} else {
				 //如果超过了设置的频率限制，本次onDemandUpdate方法就提交任务了
				logger.warn("Ignoring onDemand update due to stopped scheduler");
				return false;
			}
		} else {
			logger.warn("Ignoring onDemand update due to rate limiter");
			return false;
		}
	}

	/**
	* 关键代码：将服务信息注册到eureka server的实现
	*/
	public void run() {
		try {
			// 刷新本地实例配置信息，如果本地配置信息发生了变化则调用com.netflix.appinfo.InstanceInfo#setIsDirty()将当前实例状态改为脏数据状态，以便下一步判断是否发生实例状态变化将实例信息注册到服务端
			discoveryClient.refreshInstanceInfo();
			// 当前任务启动时start()方法会将实例信息状态设置为脏数据状态
			// 判断实例信息是否发生变化
			Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
			if (dirtyTimestamp != null) {
				// 关键代码：服务注册，将实例信息注册到eureka server
				discoveryClient.register();
				// 清除脏数据状态
				instanceInfo.unsetIsDirty(dirtyTimestamp);
			}
		} catch (Throwable t) {
			logger.warn("There was a problem with the instance info replicator", t);
		} finally {
			Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
			scheduledPeriodicRef.set(next);
		}
	}
}
```

- **EurekaDiscoveryClientConfiguration （DiscoveryClient自动化配置）**

spring cloud commons与eureka集成的自动化配置核心类

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@ConditionalOnBlockingDiscoveryEnabled
public class EurekaDiscoveryClientConfiguration {
	/**
	* 最最关键的配置：服务发现的具体实现bean（spring cloud commons项目中DiscoveryClient的具体实现）
	* 这里注入的EurekaClient和EurekaClientConfig参数都是在EurekaClientAutoConfiguration中实例化bean的，
	*
	* 该类的作用是适配spring cloud commons中的DiscoveryClient与eureka， 起到一个桥梁作用，
	* 本身EurekaDiscoveryClient中的代码非常简洁，都是调用了netflix自身的EurekaClient, 所有关键的服务发现、
	* 服务注册、心跳都是在EurekaClient的构造方法中实现的（启用了一系列的定时任务、注册服务状态变更监听器）
	*/
	@Bean
	@ConditionalOnMissingBean
	public EurekaDiscoveryClient discoveryClient(EurekaClient client,
			EurekaClientConfig clientConfig) {
		return new EurekaDiscoveryClient(client, clientConfig);
	}
	
	/**
	* 监听了RefreshScopeRefreshedEvent事件
	*/
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(RefreshScopeRefreshedEvent.class)
	protected static class EurekaClientConfigurationRefresher
			implements ApplicationListener<RefreshScopeRefreshedEvent> {

		/**
		* netflix原生的服务注册、获取服务列表等操作实现
		*/
		@Autowired(required = false)
		private EurekaClient eurekaClient;
		
		/**
		* 进行实例自动注册到注册中心的处理逻辑实现（spring cloud commons项目AutoServiceRegistration的实现）
		*/
		@Autowired(required = false)
		private EurekaAutoServiceRegistration autoRegistration;

		public void onApplicationEvent(RefreshScopeRefreshedEvent event) {
			// This will force the creation of the EurkaClient bean if not already created
			// to make sure the client will be reregistered after a refresh event
			if (eurekaClient != null) {
				eurekaClient.getApplications();
			}
			if (autoRegistration != null) {
				// register in case meta data changed
				this.autoRegistration.stop();
				this.autoRegistration.start();
			}
		}

	}
}
```

- **EurekaRegistration 实例信息， Registration的实现**

实现了spring cloud commons项目中的`Registration`接口，代表要注册的实例信息
```java
public class EurekaRegistration implements Registration {
	private final EurekaClient eurekaClient;

	private final AtomicReference<CloudEurekaClient> cloudEurekaClient = new AtomicReference<>();

	private final CloudEurekaInstanceConfig instanceConfig;

	private final ApplicationInfoManager applicationInfoManager;

	private ObjectProvider<HealthCheckHandler> healthCheckHandler;
	
	/**
	* 获取实例id, 通过调用instanceConfig实现
	* 由EurekaClientAutoConfiguration中的EurekaInstanceConfigBean实例化设置的默认值为 
	* ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id}或${server.port}
	*/
	@Override
	public String getInstanceId() {
		return this.instanceConfig.getInstanceId();
	}

	/**
	* 获取服务id, 通过调用instanceConfig实现，值为${spring.application.name}
	* 来源于org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean#setEnvironment中的setAppname方法
	*/
	@Override
	public String getServiceId() {
		return this.instanceConfig.getAppname();
	}

	/**
	* 获取实例主机名，来源于EurekaInstanceConfigBean的构造方法中的this.hostname = this.hostInfo.getHostname();赋的值
	*/
	@Override
	public String getHost() {
		return this.instanceConfig.getHostName(false);
	}
	
	/**
	* 获取实例端口号，来源于org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration#eurekaInstanceConfigBean中的instance.setNonSecurePort(serverPort);
	*/
	@Override
	public int getPort() {
		if (this.instanceConfig.getSecurePortEnabled()) {
			return this.instanceConfig.getSecurePort();
		}
		return this.instanceConfig.getNonSecurePort();
	}

	/**
	* 是否安全连接，来源类同上，由instance.setSecurePortEnabled(isSecurePortEnabled);方法设置值的
	*/
	@Override
	public boolean isSecure() {
		return this.instanceConfig.getSecurePortEnabled();
	}

	/**
	* 调用的是spring cloud commons项目中方法
	* String uri = String.format("%s://%s:%s", scheme, instance.getHost(),
				instance.getPort()); 
	*/
	@Override
	public URI getUri() {
		return DefaultServiceInstance.getUri(this);
	}

	@Override
	public Map<String, String> getMetadata() {
		return this.instanceConfig.getMetadataMap();
	}


}
```

# 总结spring-cloud-nextflix-eureka-client启动流程：
1.  `@EnableDiscoveryClient`引入`EnableDiscoveryClientImportSelector`
2. `spring-cloud-netflix-eureka-client-2.2.0.RELEASE.jar!\META-INF\spring.factories`中会自动化配置`EurekaClientConfigServerAutoConfiguration`、`EurekaDiscoveryClientConfigServiceAutoConfiguration`、`EurekaClientAutoConfiguration`、`EurekaDiscoveryClientConfiguration`、`RibbonEurekaAutoConfiguration`等几个类
3. `EurekaClientAutoConfiguration`   会实例化如下几个类：
- `EurekaInstanceConfigBean`   
读取application.yml中eureka.instance为前缀的配置
- `EurekaServiceRegistry`   
spring cloud commons项目ServiceRegisity接口的实现
- `EurekaAutoServiceRegistration`   
spring cloud commons项目AutoServiceRegistration接口的实现
- `EurekaClient`   
netfliex的服务发现、服务注册、心跳实现，**构造方法**中会发送第一次rest请求，全量获取所有服务注册信息，然后启动一系列定时任务（心跳、刷新服务发现信息、实例状态变化时注册注册实例信息）
- `ApplicationInfoManager`
持有实例信息
- `EurekaRegistration`
spring cloud commons项目Registration接口的实现