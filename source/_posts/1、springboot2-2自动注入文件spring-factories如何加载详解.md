---
title: 1、springboot2.2自动注入文件spring.factories如何加载详解
date: 2019-12-29 16:50:34
tags: spring boot
categories: spring boot
---

# 1.首先看下启动类：
```java
@SpringBootApplication
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class);
  	}
}
```

启动类使用```@SpringBootApplication```注解，再看下这个注解内容:
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
	
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};
	
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};
	
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};
	
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};
	
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;
}
```

再进入```@EnableAutoConfiguration```中查看
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	...省略
}
```

发现加载了一个```AutoConfigurationImportSelector.class```类

温馨提示：所有实现ImportSelector的类，都会在启动时被```org.springframework.context.annotation.ConfigurationClassParser```中的```processImports```进行实例化，并执行```selectImports```方法。

# 2、AutoConfigurationImportSelector中的selectImports
```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

	private static final AutoConfigurationEntry EMPTY_ENTRY = new AutoConfigurationEntry();

	private static final String[] NO_IMPORTS = {};

	private static final String PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = "spring.autoconfigure.exclude";

	private ConfigurableListableBeanFactory beanFactory;

	private Environment environment;

	private ClassLoader beanClassLoader;

	private ResourceLoader resourceLoader;
			
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		/** 1、入口关键代码**/
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
				annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
			
	protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		
		/** 2、获取所有jar包中的spring.factories文件中的key为EnableAutoConfiguration的值*/
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		
		/** 移除重复的配置值 **/
		configurations = removeDuplicates(configurations);
		/** 获取注解上配置的要排序的class的全限定name*/
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		/**移除需要排除的自动配置类*/
		configurations.removeAll(exclusions);
		/**过滤自动配置类*/
		configurations = filter(configurations, autoConfigurationMetadata);
		/**触发事件，通知相关监听该事件的listener*/
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
			
	protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass() == AutoConfigurationImportSelector.class) {
			return getEnvironment().getProperty(EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class, true);
		}
		return true;
	}
			
	protected Class<?> getAnnotationClass() {
		return EnableAutoConfiguration.class;
	}
			
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		/** 3、这里的SpringFactoriesLoader.loadFactoryNames是重点*/
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
	

	protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		/**要自动化配置的key*/
		return EnableAutoConfiguration.class;
	}
```
**从上述代码可以看到获取自动化配置相关的类最终是调用```SpringFactoriesLoader.loadFactoryNames(Class cls, ClassLoader classLoader);```实现的**

# 3、 SpringFactoriesLoader如何加载

```java
public final class SpringFactoriesLoader {
	/**spring.factories的位置*/
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

	private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
	/**
	* 缓存扫描后的结果， 注意这个cache是static修饰的，说明是多个实例共享的
	* 其中MultiValueMap的key就是spring.factories中的key（比如org.springframework.boot.autoconfigure.EnableAutoConfiguration）, 
	* 其值就是key对应的value以逗号分隔后得到的List集合（这里用到了MultiValueMap，他是guava的一种多值map， 类似Map<String, List<String>>）
	*/
	private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();

	private SpringFactoriesLoader() {
	}
	
	/**
	* AutoConfigurationImportSelector里最终调用的就是这个方法，
	* 这里的factoryType是EnableAutoConfiguration.class，
	* classLoader是AutoConfigurationImportSelector里的beanClassLoader
	*/
	public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
		String factoryTypeName = factoryType.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
	}
	
	/**
	* 加载 spring.factories文件的核心实现
	*/
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		// 先从缓存获取，如果获取到了说明之前已经被加载过
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			// 找到所有jar中的spring.factories文件的地址
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			// 循环处理每一个spring.factories文件
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				// 加载spring.factories文件中的内容到Properties对象中
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				// 遍历spring.factories内容中的所有的键值对
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					// 获得spring.factories内容中的key（比如org.springframework.boot.autoconfigure.EnableAutoConfiguratio）
					String factoryTypeName = ((String) entry.getKey()).trim();
					// 获取value， 然后按英文逗号(,)分割得到value数组并遍历
					for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						// 存储结果到上面的多值Map中(MultiValueMap<String, String>)
						result.add(factoryTypeName, factoryImplementationName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
}
```

# 4、spring.factories加载过程总结
1. 我们自己会使用```@SpringBootApplication```、```@EnableDiscoryClient```等注解
2. 这些注解一般会再引入```@Import(AutoConfigurationImportSelector.class)```这样的类  
3. ```AutoConfigurationImportSelector```这样的Selector会执行其中的```String[] selectImports(AnnotationMetadata annotationMetadata)```方法来加载```spring.factories```中的自动化配置，方法内部会调用```SpringFactoriesLoader
				.loadFactoryNames(Class factoryType, this.beanClassLoader)```来**真正加载spring.factories**
4. ```SpringFactoriesLoader.loadFactoryNames(Class factoryType, this.beanClassLoader)```**扫描所有jar**中的META-INF/spring.factories文件，将内容存储到一个```MultiValueMap<String, String>```中缓存起来，下次加载时直接从缓存中找  