---
title: spring-cloud-commons 源码分析
date: 2019-12-29 16:47:48
tags: spring cloud
categories: spring cloud
---



SpringCloud组件内部一定会有```spring-cloud-commons``` 和 ```spring-cloud-context``` 这两个依赖中的一个。比如 ```spring-cloud-netflix-eureka-server```, ```spring-cloud-netflix-eureka-client```, ```spring-cloud-netflix-ribbon```。它们内部的这两个依赖都是optional。这些组件对应的starter内部使用了``` spring-cloud-starter``` 依赖，因为```spring-cloud-starter```依赖内部依赖了```spring-cloud-context```、```spring-cloud-commons```和```spring-boot-starter```(springboot全套架构)。

本文将分析```spring-cloud-commons```模块。关于```spring-cloud-context```将在下一篇文章中分析。

```spring-cloud-commons```模块是spring在分布式领域上(服务发现，服务注册，断路器，负载均衡)的规范定义(```spring-cloud-netflix```是具体的实现，也就是```Netflix OSS```里的各种组件实现了这个commons规范)，可以被所有的Spring Cloud客户端使用(比如服务发现领域的```eureka```，```consul```)。下面将根据包名来分析一下内部的一些接口和类。

# 1、actuator功能
actuator子包里提供了一个```id```为 ```features``` 的 ```FeaturesEndpoint```。该Endpoint里会展示应用具体的feature。具体的内容在HasFeatures集合属性中，```HasFeatures```内部包含 ```List<Class<?>> abstractFeatures``` 和 ```List<NamedFeature> namedFeatures``` 这两个属性。
```java
@Endpoint(id = "features")
public class FeaturesEndpoint implements ApplicationContextAware {

	private final List<HasFeatures> hasFeaturesList;

	private ApplicationContext context;

	public FeaturesEndpoint(List<HasFeatures> hasFeaturesList) {
		this.hasFeaturesList = hasFeaturesList;
	}
	
	@ReadOperation
	public Features features() {
		Features features = new Features();

		for (HasFeatures hasFeatures : this.hasFeaturesList) {
			List<Class<?>> abstractFeatures = hasFeatures.getAbstractFeatures();
			if (abstractFeatures != null) {
				for (Class<?> clazz : abstractFeatures) {
					addAbstractFeature(features, clazz);
				}
			}

			List<NamedFeature> namedFeatures = hasFeatures.getNamedFeatures();
			if (namedFeatures != null) {
				for (NamedFeature namedFeature : namedFeatures) {
					addFeature(features, namedFeature);
				}
			}
		}

		return features;
	}
	
	private void addAbstractFeature(Features features, Class<?> type) {
		String featureName = type.getSimpleName();
		try {
			Object bean = this.context.getBean(type);
			Class<?> beanClass = bean.getClass();
			addFeature(features, new NamedFeature(featureName, beanClass));
		}
		catch (NoSuchBeanDefinitionException e) {
			features.getDisabled().add(featureName);
		}
	}

	private void addFeature(Features features, NamedFeature feature) {
		Class<?> type = feature.getType();
		features.getEnabled()
				.add(new Feature(feature.getName(), type.getCanonicalName(),
						type.getPackage().getImplementationVersion(),
						type.getPackage().getImplementationVendor()));
	}

	static class Features {

		final List<Feature> enabled = new ArrayList<>();

		final List<String> disabled = new ArrayList<>();

		public List<Feature> getEnabled() {
			return this.enabled;
		}

		public List<String> getDisabled() {
			return this.disabled;
		}

	}
}	
```

```NamedFeature```里有 ```String name``` 和 ```Class<?> type``` 这两个属性。

- ```abstractFeatures```属性处理过程：遍历```abstractFeatures```集合，在```ApplicationContext```中找出具体Class的bean。然后根据这个具体的Class构造```Feature```(Feature拥有type，name，version和vendor属性)。

- namedFeatures属性处理过程：遍历namedFeatures集合，直接根据NamedFeature李的name和type构造Feature。

比如```CommonsClientAutoConfiguration```里就使用如下方式构造了一个```HasFeatures```。这个```HasFeatures```只有abstractFeatures属性有值，对应的Class是DiscoveryClient和LoadBalancerClient：
```java
@Configuration(proxyBeanMethods = false)
public class CommonsClientAutoConfiguration {
 	... 省略无关代码
		
	protected static class DiscoveryLoadBalancerConfiguration {
		... 省略无关代码
				
		@Bean
		public HasFeatures commonsFeatures() {
			return HasFeatures.abstractFeatures(DiscoveryClient.class,
					LoadBalancerClient.class);
		}
	}
}
```

下面就是一个```FeaturesEndpoint```内容，对应```DiscoveryClient```和```LoadBalancerClient```接口，具体的type就是```CompositeDiscoveryClient```和```RibbonLoadBalancerClient```：
```json
{
	enabled: [{
			type: "org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClient",
			name: "DiscoveryClient",
			version: "1.3.3.RELEASE",
			vendor: "Pivotal Software, Inc."
		},
		{
			type: "org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient",
			name: "LoadBalancerClient",
			version: "1.4.4.RELEASE",
			vendor: "Pivotal Software, Inc."
		},
		{
			type: "com.netflix.ribbon.Ribbon",
			name: "Ribbon",
			version: "2.2.5",
			vendor: null
		},
		{
			type: "rx.Observable",
			name: "MVC Observable",
			version: "1.2.0",
			vendor: null
		},
		{
			type: "rx.Single",
			name: "MVC Single",
			version: "1.2.0",
			vendor: null
		}
	],
	disabled: []
}
```

# ２、circuitbreaker功能
断路器功能。

```circuitbreaker```子包里面定义了一个注解```@EnableCircuitBreaker```和一个Import Selector。只要使用了该注解就会import这个selector：
```
@Import(EnableCircuitBreakerImportSelector.class)
public @interface EnableCircuitBreaker {
	...
}
```

selector也很简单，代码如下：
```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableCircuitBreakerImportSelector extends
		SpringFactoryImportSelector<EnableCircuitBreaker> {
	@Override
	protected boolean isEnabled() {
		return getEnvironment().getProperty(
				"spring.cloud.circuit.breaker.enabled", Boolean.class, Boolean.TRUE);
	}
}
```
继承了``` SpringFactoryImportSelector```， 内部会使用工厂加载机制。去加载```META-INF/spring.factories```里key为 ```org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker ```的类。该机制生效的前期是 ```spring.cloud.circuit.breaker.enabled ```配置为true，默认值就是true。

```circuitbreaker```子包相当于了定义了断路器的加载机制。在```spring.factories```里配置对应的类和开关配置即可生效。具体的实现由其它模块提供。

# 3、discovery功能
## 3.1、服务发现功能

定义了```DiscoveryClient```接口和```EnableDiscoveryClient```注解。

定义了一些各种服务发现组件客户端里的读取服务操作：
```java
public interface DiscoveryClient {
	String description(); // 描述
	
	List<ServiceInstance> getInstances(String serviceId); // 根据服务id获取具体的服务实例
	
	List<String> getServices(); // 获取所有的服务id集合
}
```

```@EnableDiscoveryClient```注解import了```EnableDiscoveryClientImportSelector```这个selector。该注解内部有个属性 ```boolean autoRegister() default true; ```表示是否自动注册，默认是true。

selector内部会找出 ```META-INF/spring.factories```里key为```org.springframework.cloud.client.discovery.EnableDiscoveryClient```的类。

如果自动注册属性为```true```，会在找出的这些类里再加上一个类：```org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration```。 ```AutoServiceRegistrationConfiguration```内部会使用```@EnableConfigurationProperties(AutoServiceRegistrationProperties.class)```触发构造```AutoServiceRegistrationProperties```这个bean。像eureka，nacos，它们的自动化配置类里都使用了```@ConditionalOnBean(AutoServiceRegistrationProperties.class)```来确保存在```AutoServiceRegistrationProperties```这个bean存在的时候才会构造AutoServiceRegistration进行注册。

如果自动注册属性为false，在Environment里加一个PropertySource，内部的配置项是```spring.cloud.service-registry.auto-registration.enabled```，值是false(代表不构造```AutoServiceRegistrationProperties.class)```。这样eureka，nacos都不会注册


## 3.2、Health Indicator
springboot 中提供了一个健康检查的接口```HealthIndicator```, ```DiscoveryClient```能够通过实现```DiscoveryHealthIndicator```来做健康检查。设置```spring.cloud.discovery.client.composite-indicator.enabled=false```来禁用这种混和的健康检查；```DiscoveryClientHealthIndicator```通常是自动配置的，设置```spring.cloud.discovery.client.health-indicator.enabled=false```来禁用；设置```spring.cloud.discovery.client.health-indicator.include-description=false```来禁用description字段，如果没有禁用，就会一直向上层传递。

## 3.3、Ordering DiscoveryClient instances
```DiscoveryClient```继承了```Ordered```; 当你使用多个服务发现的时候这个会很有用，可以定义通过这种方式来按照指定的顺序来从注册中心加载bean。```DiscoveryClient```默认的order设置的是 0 ；如果想要为你自己实现的```DiscoveryClient```设置不同的order，仅仅需要覆盖```getOrder()```。除此之外Spring Cloud还提供了配置```spring.cloud.{clientIdentifier}.discovery.order```来设置order，这其中主要的实现有```ConsulDiscoveryClient```, ```EurekaDiscoveryClient```, ```ZookeeperDiscoveryClient```；

## 3.4、discovery子包内部还有其它一些功能：

- simple

简单的服务发现实现类 ```SimpleDiscoveryClient```，具体的服务实例从 ```SimpleDiscoveryProperties``` 配置中获取。 SimpleDiscoveryProperties 配置 读取前缀为 ```spring.cloud.discovery.client.simple``` 的配置。读取的结果放到Map里 ```Map<String, List<SimpleServiceInstance>>```。这里 ```SimpleServiceInstance``` 实现了```ServiceInstance```接口。 具体的属性值从 ```SimpleDiscoveryProperties``` 中获取

```SimpleDiscoveryClientAutoConfiguration``` 自动化配置类在 spring.factories里key为 ```org.springframework.boot.autoconfigure.EnableAutoConfiguration```的配置项中。内部会构造 ```SimpleDiscoveryProperties、 SimpleDiscoveryClient```

- noop

什么都不做的服务发现实现类，已经被废弃，建议使用 simple 子模块里的类代替

- health

SpringBoot的那套health机制与SpringCloud结合。使用DiscoveryClient获取服务实例的信息

- event

定义了一些心跳检测事件，服务注册事件

- composite

定义了 ```CompositeDiscoveryClient```。 看名字也知道，组合各个服务发现客户端的一个客户端。默认会根据```CompositeDiscoveryClientAutoConfiguration```自动化配置类构造出```CompositeDiscoveryClient```。默认清下我们注入的```DiscoveryClient```就是这个```CompositeDiscoveryClient```

# 4、serviceregistry功能
## 4.1、ServiceRegistry服务注册功能

定义了服务注册的接口 ```ServiceRegistry<R extends Registration>```，Registration接口继承了服务实例```ServiceInstance```接口，未新增新方法，留作以后扩展使用。
```java
public interface ServiceRegistry<R extends Registration> {
	// 注册服务实例
	void register(R registration);
	// 下线服务实例
	void deregister(R registration);
	// 生命周期方法，关闭操作
	void close();
	// 设置服务实例的状态，状态值由具体的实现类决定
	void setStatus(R registration, String status);
	// 获取服务实例的状态
	<T> T getStatus(R registration);
}
```

比如要取消自动注册，改为手动注册，示例代码如下：
```java
@Configuration
// 这里autoRegister属性设置为false表示取消自动注册
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {
	// 服务注册接口，其实现是一个
	private ServiceRegistry registry;

	public MyConfiguration(ServiceRegistry registry) {
		this.registry = registry;
	}

 	// called through some external process, such as an event or a custom actuator endpoint
	public void register() {
		// 要注册的服务实例信息
 		Registration registration = constructRegistration();
		
		// 由底层的服务注册实现去注册
 		this.registry.register(registration);
  	}
}
```
每个```ServiceRegistry```实现类都会提供一个对应的服务注册实现

- ```ZookeeperRegistration``` 使用的 ```ZookeeperServiceRegistry```  
- ```EurekaRegistration``` 使用的 ```EurekaServiceRegistry```
- ```ConsulRegistration``` 使用的 ```ConsulServiceRegistry```

## 4.2、ServiceRegistry Auto-Registration
### 4.2.1、禁用自动注册服务功能 
默认情况下ServiceRegistry的实现类在运行的时候会自动注册服务，两种方式来禁用自动注册服务
```
# 通过注解方式禁用自动注册服务
@EnableDiscoveryClient(autoRegister=false)

# 通过配置yml属性配置方式禁用自动注册服务
spring.cloud.service-registry.auto-registration.enabled=false
```
当一个服务自动注册的时会触发两个事件：  
- 第一个是```InstancePreRegisteredEvent```，在注册之前触发；  
- 第二个是```InstanceRegisteredEvent``` 在注册完成之后触发；可以使用```ApplicationListener```来监听这两个事件

当```spring.cloud.service-registry.auto-registration.enabled=false```的时候就不会触发这两个事件

### 4.2.2、原理剖析
spring-cloud-commons项目的```serviceresgistry```包中定义了一些自动化配置类：

- ```ServiceRegistryAutoConfiguration```  
内部会根据条件注解判断是否构造ServiceRegistryEndpoint，该endpoint会暴露ServiceRegistry的状态信息，也可以设置ServiceRegistry的状态信息。  

  Spring Cloud Commons 提供了一个```/service-registry``` 端点，这个endpoint依赖于容器中的```Registration```。GET请求这个地址将会返回```Registration```的状态；POST请求这个地址可以修改Registration,这个json格式的body中必须要包含一个status;查询```ServiceRegistry```的实现类文档来确定status的值；比如Eureka的状态值：UP, DOWN, OUT_OF_SERVICE, UNKNOWN.

- ```AutoServiceRegistrationAutoConfiguration```

在```@EnableDiscoveryClient```注解打开自注册开关的时候才会生效，内部import了```AutoServiceRegistrationConfiguration```这个类，该类内部会使用```@EnableConfigurationProperties```注解构造```AutoServiceRegistrationProperties```这个bean

- 定义了一个接口AutoServiceRegistration 和一个抽象类```AbstractAutoServiceRegistration```，用于处理服务自动注册逻辑。一般我们自定义的服务注册逻辑只需要继承该类即可。

```AutoServiceRegistration```接口无任何方法声明，用于标记是否是服务自动注册。

```AbstractAutoServiceRegistration``` 抽象类实现了```AutoServiceRegistration```接口，定义了4个抽象方法：
```
// 服务注册信息的配置数据
protected abstract Object getConfiguration();
// 该注册器是否可用
protected abstract boolean isEnabled();
// 获取注册信息
protected abstract R getRegistration();
// 获取注册信息的management信息
protected abstract R getManagementRegistration();
```

**```AbstractAutoServiceRegistration``` 抽象类内部逻辑总结：**

1. 构造方法里必须有个```ServiceRegistry```参数，服务注册相关的逻辑都使用该接口完成

2. 监听```WebServerInitializedEvent```事件。
当WebServer初始化完毕后(Spring ApplicationContext也已经refresh后)，使用```ServiceRegistry```注册服务，具体的服务信息在抽象方法getRegistration()里由子类实现。当子类实现的getManagementRegistration()接口有返回具体的注册信息并且配置的management信息后注册这个management信息

3. 该类销毁的时候使用```ServiceRegistry```下线服务(下线过程跟注册过程雷同，下线```getRegistration()```和
```getManagementRegistration()```方法里返回的注册信息)，并调用```ServiceRegistry```的close方法关闭注册器。

**总结一下，在SpringCloud体系下要实现新的服务注册、发现需要这6个步骤(最新版本的spring-cloud-commons已经建议我们直接使用Registration，废弃ServiceInstance)：**

1. 实现```ServiceRegistry```接口，完成服务注册自身的具体逻辑
2. 实现```Registration```接口，完成服务注册过程中获取注册信息的操作
3. 继承```AbstractAutoServiceRegistration```，完成服务注册前后的逻辑
4. 实现```DiscoveryClient```接口，完成服务发现的具体逻辑
5. 实现```ServiceInstance```接口，在```DiscoveryClient```接口中被使用，完成服务注册组件与SpringCloud注册信息的转换
自动化配置类，将这些Bean进行构造

# 5、loadbalancer功能
## 5.1、使用示例
创建一个支持负载均衡的```RestTemplate```，使用```@LoadBalanced```和```@Bean```注解，像下面的例子：
```
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        String results = restTemplate.getForObject("http://stores/stores", String.class);
        return results;
    }
}
```

## 5.2、客户端负载均衡功能


一些接口的定义：

- ```ServiceInstanceChooser```：服务实例选择器，使用load balancer根据serviceId获取具体的实例。
```		
public interface ServiceInstanceChooser {
	// 根据服务id获取具体的服务实例
    ServiceInstance choose(String serviceId);
}
```

- ```LoadBalancerClient```：负载均衡客户端，继承```ServiceInstanceChooser```。
```
public interface LoadBalancerClient extends ServiceInstanceChooser {
	// 根据serviceId使用ServiceInstance执行请求
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	// 重载方法。同上
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
	// 基于服务实例和URI重新构造一个新的带有host和port信息的URI。比如http://myservice/path/to/service.这个地址 myservice 这个服务名将会替换成比如 192.168.1.122:8080
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

- ```RestTemplateCustomizer```：RestTemplate的定制化器。
```
public interface RestTemplateCustomizer {
	void customize(RestTemplate restTemplate);
}
```

- ```LoadBalancerRequestTransformer```：HttpRequest转换器，根据ServiceInstance转换成一个新的具有load balance功能的HttpRequest。
```
@Order(LoadBalancerRequestTransformer.DEFAULT_ORDER)
public interface LoadBalancerRequestTransformer {
	public static final int DEFAULT_ORDER = 0;
	HttpRequest transformRequest(HttpRequest request, ServiceInstance instance);
}
```
- ```LoadBalancerRequest```：函数式接口。对ServiceInstance操作并返回具体的泛型T。```LoadBalancerRequestFactory的createRequest```方法内部实现了该接口。实现过程中使用```LoadBalancerRequestTransformer```对request进行转换并返回了ClientHttpResponse。
```
public interface LoadBalancerRequest<T> {
	public T apply(ServiceInstance instance) throws Exception;
}
```

- ```@LoadBalanced```注解用于修饰RestTemplate，表示使用负载均衡客户端。

使用该注解修饰的RestTemplate会在```LoadBalancerAutoConfiguration```自动化配置类中被处理：
```
// 使用RestTemplateCustomizer定制化这些被@LoadBalanced注解修饰的RestTemplate
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
        final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
    return () -> restTemplateCustomizers.ifAvailable(customizers -> {
        for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
            for (RestTemplateCustomizer customizer : customizers) {
                customizer.customize(restTemplate);
            }
        }
    });
}

// 内部的一个配置类LoadBalancerInterceptorConfig
// 如果没有依赖spring-retry模块。如果依赖spring-retry模块的话会构造另外一个配置类RetryInterceptorAutoConfiguration。
// 内部也会1个拦截器和1个定制化器，分别是RetryLoadBalancerInterceptor和RetryLoadBalancerInterceptor。原理类似

@Configuration
@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
static class LoadBalancerInterceptorConfig {
	// 构造http拦截器
    @Bean
    public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory) {
        return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
    }
	// 构造定制化器RestTemplateCustomizer，为这些RestTemplate添加拦截器LoadBalancerInterceptor
    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(
            final LoadBalancerInterceptor loadBalancerInterceptor) {
        return restTemplate -> {
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                    restTemplate.getInterceptors());
            list.add(loadBalancerInterceptor);
            restTemplate.setInterceptors(list);
        };
    }
}
```
```LoadBalancerInterceptor```拦截器内部会对request请求进行拦截。拦截器内部使用LoadBalancerClient完成请求的调用，这里调用的时候需要的```LoadBalancerRequest```由```LoadBalancerRequestFactory```构造，```LoadBalancerRequestFactory```内部使用```LoadBalancerRequestTransformer```对request进行转换。

## 5.3 Retrying Failed Requests
```RestTemplate```可以配置请求失败后的重试策略；默认这个逻辑是禁止的，如果需要可以开启，只需要添加 Spring Retry到classpath; 如果spring retry已经在classpath，你想要禁用这个retry的功能，那么可以配置```spring.cloud.loadbalancer.retry.enabled=false```

如果想要自定义一个```BackOffPolicy```,需要创建一个```LoadBalancedRetryFactory```并覆写方法```createBackOffPolicy```; eg:
```
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryFactory retryFactory() {
        return new LoadBalancedRetryFactory() {
            @Override
            public BackOffPolicy createBackOffPolicy(String service) {
                return new ExponentialBackOffPolicy();
            }
        };
    }
}
```
## 5.4 Multiple RestTemplate objects
如何创建一个支持负载均衡的```RestTemplate```和不支持负载均衡的```RestTemplate```以及注入的方式？看下面的列子：
```
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate loadBalanced() {
        return new RestTemplate();
    }

    @Primary
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    @LoadBalanced
    private RestTemplate loadBalanced;

    public String doOtherStuff() {
        return loadBalanced.getForObject("http://stores/stores", String.class);
    }

    public String doStuff() {
        return restTemplate.getForObject("http://example.com", String.class);
    }
}
```
```@Primary```的作用是在使用```@Autowired```注入时，如果发现了多个类型的bean, 就选择使用了```@Primary```的bean

如果遇到了这个异常```java.lang.IllegalArgumentException: Can not set org.springframework.web.client.RestTemplate field com.my.app.Foo.restTemplate to com.sun.proxy.$Proxy89```,可以尝试注入类型修改```RestOperations```或者设置```spring.aop.proxyTargetClass=true```

# 6、hypermedia功能
springcloud对```hateoas```在服务发现领域上的支持。

关于hateoas可以参考一些资料：

https://github.com/spring-projects/spring-hateoas

https://spring.io/guides/gs/rest-hateoas/

## 其它注解、接口、类
- @SpringCloudApplication注解

整合了@SpringBootApplication、@EnableDiscoveryClient、@EnableCircuitBreaker这3个注解，说明@SpringCloudApplication注解表示一个分布式应用注解

- ```ServiceInstance```接口

表示服务发现系统里的一个服务实例

- ```DefaultServiceInstance```类

实现了ServiceInstance接口，是个默认的服务实例的实现

- ```HostInfoEnvironmentPostProcessor```类

属于```EnvironmentPostProcessor```。这个postprocessor会在`Environment里加上当前vm的hostname和ip信息

- ```CommonsClientAutoConfiguration```类

自动化配置类。在该模块的 ```META-INF/spring.factories```里配置，key为```org.springframework.boot.autoconfigure.EnableAutoConfiguration```。所以默认会被加载，内部会构造一些HealthIndicator，一些Endpoint

Spring Cloud Alibaba内部的```spring-cloud-alibaba-nacos-discovery```模块实现了```spring-cloud-commons```规范，提供了基于Nacos的服务发现，服务注册功能。Nacos是一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。