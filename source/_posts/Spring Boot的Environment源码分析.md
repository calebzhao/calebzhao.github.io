---
title: Spring Boot的Environment源码分析
date: 2019-12-31 09:32:37
tags: 
- spring boot
- envionment
categories: spring boot
---



# 前言

`org.springframework.core.env.Environment`是当前应用运行环境的公开接口，主要包括应用程序运行环境的两个关键方面：配置文件(profiles)和属性(properties)。`Environment`继承自接口`PropertyResolver`，而`PropertyResolver`提供了属性访问的相关方法。这篇文章从源码的角度分析`Environment`的存储容器和加载流程，然后基于源码的理解给出一个生产级别的扩展。

# 哪里创建的Environment?

学习过springboot的都知道，在Springboot的main入口函数中调用SpringApplication.run(DemoApplication.class,args)函数便可以启用SpringBoot应用程序，跟踪一下SpringApplication源码可以发现，最终还是调用了SpringApplication的动态run函数。



我们在编写一个spring boot应用时通常启动的方式是通过`SpringApplication.run(xxx.class, args)`来启动的，

```java
@SpringBootApplication
public class ClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

```SpringApplication```源码如下：

```java
public class SpringApplication{

    // ...省略与Environment无关的代码

    public SpringApplication(Class<?>... primarySources) {
        this(null, primarySources);
    }

    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class<?>[] { primarySource }, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return new SpringApplication(primarySources).run(args);
    }

    public ConfigurableApplicationContext run(String... args) {
        // ...省略与Environment无关的代码

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

            // 与Environment相关的关键代码
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

            ...省略与Environment无关的代码
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        // ...省略与Environment无关的代码
    }

    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                       ApplicationArguments applicationArguments) {
        // 关键代码：获取ConfigurableEnvironment， 如果不存在就创建
        ConfigurableEnvironment environment = getOrCreateEnvironment();
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

    // 根据应用类型创建相应的ConfigurableEnvironment
    private ConfigurableEnvironment getOrCreateEnvironment() {
        if (this.environment != null) {
            return this.environment;
        }
        switch (this.webApplicationType) {
            case SERVLET:
                // servlet应用
                return new StandardServletEnvironment();
            case REACTIVE:
                // reactive反应式应用
                return new StandardReactiveWebEnvironment();
            default:
                // 非reactive， 非web， 也就是标准java应用
                return new StandardEnvironment();
        }
    }

    // ...省略与Environment无关的代码
}

```

可以看到`SpringApplication`类中最终会根据应用类型(`WebApplicationType`枚举类)创建相应的`ConfigurableEnvironment`的具体实现实例对象，另外关于SpringApplication这个类的具体执行流程另一篇博客已有详细的源码分析，这里不再赘述，参见[Spring Boot启动流程分析](https://calebzhao.github.io/2019/12/30/Spring%20Boot启动流程分析/)

下面以web环境的`StandardServletEnvironment`为例进行分析

# Environment类体系

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191231093757.png)

- `PropertyResolver`：用于针对任何基础源解析属性的接口，提供属性访问功能
- `ConfigurablePropertyResolver`：继承自`PropertyResolver`，额外提供属性类型转换(基于`org.springframework.core.convert.ConversionService`)功能
- `Environment`：继承自`PropertyResolver`，额外提供访问和判断profiles的功能
- `ConfigurableEnvironment`：继承自`ConfigurablePropertyResolver`和`Environment`，并且提供设置激活的profile和默认的profile的功能。
- `ConfigurableWebEnvironment`：继承自`ConfigurableEnvironment`，并且提供配置`Servlet`上下文和`Servlet`参数的功能。
- `AbstractEnvironment`：实现了`ConfigurableEnvironment`接口，默认属性和存储容器的定义，并且实现了`ConfigurableEnvironment`中的方法，并且为子类预留可重写的扩展方法。
- `StandardEnvironment`：继承自`AbstractEnvironment`，非`Servlet`(Web)环境下的标准`Environment`实现。
- `StandardServletEnvironment`：继承自`StandardEnvironment`，`Servlet`(Web)环境下的标准`Environment`实现。
- `MockEnvironment`: `ConfigurableEnvironment`的简单实现，出于测试的目的，用于暴露`setProperty(String, String)` 及 `withProperty(String, String)`方法
- ```AbstractPropertyResolver```：抽象基类，用于根据任何基础源解析属性， ```conversionService```的默认实现使用```DefaultConversionService```创建。
- ```PropertySourcesPropertyResolver```：```PropertyResolver```的实现，可以针对一组基础的```PropertySource```解析属性值

reactive相关的暂时不研究。



# Environment提供的方法

一般情况下，我们在SpringMVC项目中启用到的是`StandardServletEnvironment`，它的父接口是`ConfigurableWebEnvironment`，我们可以查看此接口提供的方法：

<img src="https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191231100908.png" style="zoom: 80%;" />



- `PropertyResolover`：从接口提供的方法可以看到`PropertyResolover`接口提供的方法都是获取属性相关的get方法或者求值(resolve)方法，不涉及set操作
- `ConfigurablePropertyResolover`：扩展了`PropertyResolover`接口提供的方法, 额外提供了访问属性转换(`ConfigurableConversionService`)的get/set方法，及属性占位符`placheholder`的get/set方法，即该类的目的是提供写操作(set)的相关方法
- `Environment`：从提供的方法可以看到全部是与profile相关的访问，获取当前默认的profile及已激活的profile， 只有get操作
- `ConfigurableEnvironment`：从类名就知道提供的对`Environment`写操作相关的方法，包括设置默认profile、设置当前激活哪个profile、获取系统属性、合并另一个`ConfigurableEnvironment`的属性
- `ConfigurableWebEnvironment`：只提供了`initPropertySources`方法, 从参数就可以看出来是提供配置`ServletContext`上下文和`ServletConfig`参数的功能

因此从每个类提供的方法可以看到Environment相关的接口的思想是一层层抽象出标准环境、web环境、可配置的标准环境、可配置的web环境，将读操作和写操作分离、web环境和非web环境分离



# Environment的存储容器

从`SpringApplication.getOrCreateEnvironment()`方法可以看到最终返回的`ConfigurableEnvironment`就是`StandardServletEnvironment` 或`StandardEnvironment`或者`StandardReactiveWebEnvironment`

```java
// 根据应用类型创建相应的ConfigurableEnvironment
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
        case SERVLET:
            // servlet应用
            return new StandardServletEnvironment();
        case REACTIVE:
            // reactive反应式应用
            return new StandardReactiveWebEnvironment();
        default:
            // 非reactive， 非web， 也就是标准java应用
            return new StandardEnvironment();
    }
}
```

下面从StandardServletEnvironment开始分析。

new StandardServletEnvironment()会执行StandardServletEnvironment的的构造方法，会发现这个类没有提供构造方法，只重写了`customizePropertySources`及`initPropertySources`这2个方法，这2个类后续分析。

我们继续看`StandardServletEnvironment`的父类`StandardEnvironment`的实现，发现`StandardEnvironment`也是非常简单，只重写了`customizePropertySources()`这个方法，暂且不管，继续看它的父类`AbstractEnvironment`，会发现这个类做了很多事情，定义了propertySources及profile相关属性，下面具体分析`AbstractEnvironment`这个类。

## PropertySource

要分析AbstractEnvironment这个类先看`PropertySource`

```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment {
    // ...省略代码

    private final MutablePropertySources propertySources = new MutablePropertySources();

    // ...省略代码

    public AbstractEnvironment() {
        customizePropertySources(this.propertySources);
    }

    protected void customizePropertySources(MutablePropertySources propertySources) {

    }
}
```

上面的`propertySource`属性就是用来存放`PropertySource`列表的，再看`MutablePropertySources`这个类的实现：

```java
public class MutablePropertySources implements PropertySources {

	private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
    
    ...省略代码
}
```

可以看到其中维护了PropertySource这个类的集合propertySourceList属性，这个就是最底层的存储容器。继续跟进看PropertySource的源码实现。

```java
public abstract class PropertySource<T> {

    protected final String name;

    protected final T source;

    public PropertySource(String name, T source) {
        Assert.hasText(name, "Property source name must contain at least one character");
        Assert.notNull(source, "Property source must not be null");
        this.name = name;
        this.source = source;
    }

    public PropertySource(String name) {
        this(name, (T) new Object());
    }

    public String getName() {
        return this.name;
    }

    public T getSource() {
        return this.source;
    }

    public boolean containsProperty(String name) {
        return (getProperty(name) != null);
    }

    @Nullable
    public abstract Object getProperty(String name);

    @Override
    public boolean equals(@Nullable Object other) {
        return (this == other || (other instanceof PropertySource &&
                                  ObjectUtils.nullSafeEquals(this.name, ((PropertySource<?>) other).name)));
    }

    @Override
    public int hashCode() {
        return ObjectUtils.nullSafeHashCode(this.name);
    }

    //省略其他方法和内部类的源码 
}
```

源码相对简单，预留了一个`getProperty`抽象方法给子类实现，**重点需要关注的是重写了的`equals`和`hashCode`方法，实际上只和`name`属性相关，这一点很重要，说明一个PropertySource实例绑定到一个唯一的name，这个name有点像HashMap里面的key**，部分移除、判断方法都是基于name属性。`PropertySource`的最常用子类是`MapPropertySource`、`PropertiesPropertySource`、`ResourcePropertySource`、`StubPropertySource`、`ComparisonPropertySource`。

- `MapPropertySource`：source指定为Map类型的`PropertySource`实现。

  ```java
  public class MapPropertySource extends EnumerablePropertySource<Map<String, Object>> {
  
      public MapPropertySource(String name, Map<String, Object> source) {
          super(name, source);
      }
  
  
      @Override
      @Nullable
      public Object getProperty(String name) {
          return this.source.get(name);
      }
  
      @Override
      public boolean containsProperty(String name) {
          return this.source.containsKey(name);
      }
  
      @Override
      public String[] getPropertyNames() {
          return StringUtils.toStringArray(this.source.keySet());
      }
  
  }
  ```

  

- `PropertiesPropertySource`：source指定为`Properties`类型的`PropertySource`实现，`PropertiesPropertySource` 继承了`MapPropertySource`，说明`Properties`这个类本身是集合`Map`的子类， 查看`Properties`类的源码可以发现`Properties`这个类继承了`Hashtable`, 而`HashTable`又实现了`Map`接口，所以`PropertiesPropertySource`是`MapPropertySource`的特殊化类型实现。

  ```java
  public class PropertiesPropertySource extends MapPropertySource {
  
      @SuppressWarnings({"rawtypes", "unchecked"})
      public PropertiesPropertySource(String name, Properties source) {
          super(name, (Map) source);
      }
  
      protected PropertiesPropertySource(String name, Map<String, Object> source) {
          super(name, source);
      }
  
      @Override
      public String[] getPropertyNames() {
          synchronized (this.source) {
              return super.getPropertyNames();
          }
      }
  }
  ```

  

- `ResourcePropertySource`：继承自`PropertiesPropertySource`，source指定为通过`Resource`实例转化为`Properties`再转换为Map实例。

  ```java
  public class ResourcePropertySource extends PropertiesPropertySource {
  
      @Nullable
      private final String resourceName;
  
      public ResourcePropertySource(String name, EncodedResource resource) throws IOException {
          super(name, PropertiesLoaderUtils.loadProperties(resource));
          this.resourceName = getNameForResource(resource.getResource());
      }
  
      // ...省略其他代码
  }
  ```

- `StubPropertySource`：`PropertySource`的一个内部类，source设置为null，实际上就是空实现。

  ```java
  public abstract class PropertySource<T> {
  
      protected final String name;
  
      protected final T source;
  
      public PropertySource(String name, T source) {
          Assert.hasText(name, "Property source name must contain at least one character");
          Assert.notNull(source, "Property source must not be null");
          this.name = name;
          this.source = source;
      }
  
      // ...省略代码
  
      public static class StubPropertySource extends PropertySource<Object> {
  
          public StubPropertySource(String name) {
              super(name, new Object());
          }
  
  
          @Override
          @Nullable
          public String getProperty(String name) {
              return null;
          }
      }
  }
  ```

  

- `ComparisonPropertySource`：继承自`ComparisonPropertySource`，所有属性访问方法强制抛出异常，作用就是一个不可访问属性的空实现。

  ```java
  public abstract class PropertySource<T> {	
      
       protected final String name;
  
  	protected final T source;
      
      ...省略代码
          
      static class ComparisonPropertySource extends StubPropertySource {
          
          private static final String USAGE_ERROR =
  				"ComparisonPropertySource instances are for use with collection comparison only";
          
          public ComparisonPropertySource(String name) {
  			super(name);
  		}
          
          @Override
  		public Object getSource() {
  			throw new UnsupportedOperationException(USAGE_ERROR);
  		}
  
  		@Override
  		public boolean containsProperty(String name) {
  			throw new UnsupportedOperationException(USAGE_ERROR);
  		}
  
  		@Override
  		@Nullable
  		public String getProperty(String name) {
  			throw new UnsupportedOperationException(USAGE_ERROR);
  		}
      }
  }
  ```

  

## AbstractEnvironment

```java
/** 
* 这个类是模版方法模式的应用，这个类Environment实现的抽象基类，
* 子类需要重写customizePropertySources(MutablePropertySources propertySources)方法，来将相关属性加入到MutablePropertySources中
* 
*/
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

    public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";

    public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";

    public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";

    protected static final String RESERVED_DEFAULT_PROFILE_NAME = "default";

    private final Set<String> activeProfiles = new LinkedHashSet<>();

    private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

    // 用来存放PropertySource列表的, MutablePropertySources类里面维护了List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>()集合
    private final MutablePropertySources propertySources = new MutablePropertySources();

    // 属性解析器，该属性解析器在根据key获取属性时会从多个属性源中找出第一个存在的属性，另外还支持解析属性占位符，如${server.port}
    // 具体源码后续分析，参见Environment属性访问源码分析章节
    private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);

    public AbstractEnvironment() {
        customizePropertySources(this.propertySources);
    }

    // 由子类重写该方法，将属性设置到propertySources中，这是个模版方法模式的应用
    protected void customizePropertySources(MutablePropertySources propertySources) {

    }

    // 默认的profile的名称集合， 也就是default
    protected Set<String> getReservedDefaultProfiles() {
        return Collections.singleton(RESERVED_DEFAULT_PROFILE_NAME);
    }

    // 返回当前激活的profile的名称有哪些
    // 如果当前activeProfiles没有元素，则先获取spring.profiles.active这个属性的值，然后保存到activeProfiles集合中
    @Override
    public String[] getActiveProfiles() {
        return StringUtils.toStringArray(doGetActiveProfiles());
    }

    // 如果当前activeProfiles没有元素，则先获取spring.profiles.active这个属性的值，多个激活的profile以字符串逗号(,)分隔， 例如dev,local
    protected Set<String> doGetActiveProfiles() {
        synchronized (this.activeProfiles) {
            if (this.activeProfiles.isEmpty()) {
                // 获取spring.profiles.active这个属性的值， 多个激活的profile以字符串逗号(,)分隔， 例如dev,local
                String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
                if (StringUtils.hasText(profiles)) {
                    // 多个profile逗号分割成数组后调用setActiveProfiles(String... profiles)保存起来
                    setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
                        StringUtils.trimAllWhitespace(profiles)));
                }
            }
            return this.activeProfiles;
        }
    }

    // 设置当前有哪些profile是激活的(会先清除原激活状态的profile名称，然后再设置新值)
    @Override
    public void setActiveProfiles(String... profiles) {
        Assert.notNull(profiles, "Profile array must not be null");
        if (logger.isDebugEnabled()) {
            logger.debug("Activating profiles " + Arrays.asList(profiles));
        }
        synchronized (this.activeProfiles) {
            // 清空当前已有的所有激活的profile名称
            this.activeProfiles.clear();
            for (String profile : profiles) {
                validateProfile(profile);
                // 保存方法参数传进来的新profile的名称
                this.activeProfiles.add(profile);
            }
        }
    }

    // 可变的PropertySources
    @Override
    public MutablePropertySources getPropertySources() {
        return this.propertySources;
    }



    // 返回类型转换实现， 在AbstractPropertyResolver类中默认实现是DefaultConversionService
    @Override
    public ConfigurableConversionService getConversionService() {
        return this.propertyResolver.getConversionService();
    }

    // 委托给了PropertySourcesPropertyResolver实现，具体源码后续分析，这里有个映像就行
    @Override
    @Nullable
    public String getProperty(String key) {
        return this.propertyResolver.getProperty(key);
    }

    // 委托给了PropertySourcesPropertyResolver实现，具体源码后续分析，这里有个映像就行
    @Override
    public String resolvePlaceholders(String text) {
        return this.propertyResolver.resolvePlaceholders(text);
    }
}
```

可以看到AbstractEnvironment这个类的目的就是提供Environment的抽象实现，实现了Environment的一些通用的方法，但是具体的PropertySource是怎么加入到MutablePropertySources的是由子类去处理的，所以MutablePropertySources肯定提供了一些add、remove的方法， 下面分析MutablePropertySources这个类的源码。

## MutablePropertySources

`MutablePropertySources`类中`propertySourceList`属性时最底层的存储容器，也就是环境属性都是存放在一个`CopyOnWriteArrayList>`实例中。

**该类的意图：Mutable的意思是可变的，所以说明该类的属性是可以进行写操作的, 聚合了多个属性源**

`MutablePropertySources`实现了`PropertySources`接口，先看些```PropertySources```接口的源码：

```java
public interface PropertySources extends Iterable<PropertySource<?>> {

    // jdk8的Stream
    default Stream<PropertySource<?>> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    // 是否包含某个属性源
    boolean contains(String name);

    // 根据属性源name获取属性源
    @Nullable
    PropertySource<?> get(String name);
}
```

注意PropertySources继承了集合的Iterable接口，说明它是可迭代的(可以使用增强的for循环遍历)

`MutablePropertySources`提供了`get(String name)`、`addFirst`、`addLast`、`addBefore`、`addAfter`、`remove`、`replace`等便捷方法，方便操作```propertySourceList```集合的元素，这里挑选`addBefore`的源码分析：

```java
public class MutablePropertySources implements PropertySources {
    // 存放PropertySource的最终容器
    private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();

    public MutablePropertySources() {
    }

    public MutablePropertySources(PropertySources propertySources) {
        this();
        // 循环添加到末尾
        for (PropertySource<?> propertySource : propertySources) {
            addLast(propertySource);
        }
    }

    // ...省略部分代码

    // 返回就是CopyOnWriteArrayList的迭代器实现
    @Override
    public Iterator<PropertySource<?>> iterator() {
        return this.propertySourceList.iterator();
    }

    // jdk8的流
    @Override
    public Stream<PropertySource<?>> stream() {
        return this.propertySourceList.stream();
    }

    /**
    * 这个方法需要注意，乍一看PropertySource.named(name)返回了一个新对象，
    * 怎么可能在propertySourceList中找到一个新创建的对象，一脸懵逼，不知道在搞什么鬼？
    *
    * 我们知道判断2个对象是否相等就是通过equals和hashcode方法来判断的。
    * 要回答这个问题就要看PropertySource的实现， 前面分析PropertySource源码时已经特别说明过PropertySource类重写了equals和hashCode方法，
    * 只要2个PropertySource的name属性相同就认为是等价的，具体代码这里就不再粘贴了，看到这里时请回看上一小节PropertySource的分析过程
    */
    @Override
    @Nullable
    public PropertySource<?> get(String name) {
        int index = this.propertySourceList.indexOf(PropertySource.named(name));
        return (index != -1 ? this.propertySourceList.get(index) : null);
    }

    // ...省略部分代码

    /**
    * 将新的propertySource插入到指定的relativePropertySourceName位置的前面
    *
    * @param relativePropertySourceName 在哪个属性源前面插入新属性p源propertySource
    * @param propertySource 要被插入的新属性源
    */
    public void addBefore(String relativePropertySourceName, PropertySource<?> propertySource) {
        // 前一个PropertySource的name指定为relativePropertySourceName时候必须和添加的PropertySource的name属性不相同
        assertLegalRelativeAddition(relativePropertySourceName, propertySource);
        // 如果要插入的属性源的名称在propertySourceList集合中已存在则先移除
        removeIfPresent(propertySource);
        // 获得要插入的位置
        int index = assertPresentAndGetIndex(relativePropertySourceName);
        // index这个位置原来是relativePropertySourceName，现在在这个位置插入，那么index这个位置变成了新插入的属性源， index后面的属性源全部向后移动了1位
        // 比如 原始数组[1, 3, 5], 在索引为1的位置插入元素2，那么就变成[1, 2, 3, 5]
        addAtIndex(index, propertySource);
    }

    // 新属性源的名称不能和要插入位置的属性源的名称相同（不能相对与它自己插入）
    protected void assertLegalRelativeAddition(String relativePropertySourceName, PropertySource<?> propertySource) {
        String newPropertySourceName = propertySource.getName();
        if (relativePropertySourceName.equals(newPropertySourceName)) {
            throw new IllegalArgumentException(
                "PropertySource named '" + newPropertySourceName + "' cannot be added relative to itself");
        }
    }

    // 如果该属性源的名称已存在，则移除该属性源
    protected void removeIfPresent(PropertySource<?> propertySource) {
        this.propertySourceList.remove(propertySource);
    }

    // 找出指定name的属性源的索引
    private int assertPresentAndGetIndex(String name) {
        int index = this.propertySourceList.indexOf(PropertySource.named(name));
        if (index == -1) {
            throw new IllegalArgumentException("PropertySource named '" + name + "' does not exist");
        }
        return index;
    }


    private void addAtIndex(int index, PropertySource<?> propertySource) {
        // 注意，这里会再次尝试移除同名的PropertySource
        removeIfPresent(propertySource);
        // 调用集合本身的add方法插入元素
        this.propertySourceList.add(index, propertySource);
    }
}
```

大多数`PropertySource`子类的修饰符都是public，可以直接使用，这里写个小demo：

```java
package com.calebzhao.test;

import org.springframework.core.env.MapPropertySource;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.env.PropertiesPropertySource;

import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
* 测试MutablePropertySources的使用
*/
public class MutablePropertySourceDemo {

    public static void main(String[] args) {
        // MapPropertySource
        Map<String, Object> data1 = new HashMap<>();
        data1.put("a", 1);
        data1.put("b", 2);
        MapPropertySource mapPropertySource = new MapPropertySource("p1", data1);

        // PropertiesPropertySource
        Properties properties = new Properties();
        properties.put("AA", 11);
        properties.put("BB", 22);
        PropertiesPropertySource propertiesPropertySource = new PropertiesPropertySource("p2", properties);


        // 可变PropertySources
        MutablePropertySources mutablePropertySources = new MutablePropertySources();
        mutablePropertySources.addLast(mapPropertySource);

        // 将p2插入到p1前面
        mutablePropertySources.addBefore("p1", propertiesPropertySource);

        System.out.println(mutablePropertySources);
    }
}
```

输出如下：

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191231153241.png)



# Environment属性访问源码分析

在分析AbstractEnvionment源码时没有分析属性的访问是如何实现，当时只是在getProperty()方法上注明：委托给了PropertySourcesPropertyResolver，源码后续分析，这里先回顾下AbstractEnvironment的访问属性相关的方法：

```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

    // ...省略部分代码

    // 用来存放PropertySource列表的, MutablePropertySources类里面维护了List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>()集合
    private final MutablePropertySources propertySources = new MutablePropertySources();

    // 属性解析器，该属性解析器在根据key获取属性时会从多个属性源中找出第一个存在的属性，另外还支持解析属性占位符，如${server.port}
    // 具体源码后续分析
    private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);

    public AbstractEnvironment() {
        customizePropertySources(this.propertySources);
    }

    // 由子类重写该方法，将属性设置到propertySources中，这是个模版方法模式的应用
    protected void customizePropertySources(MutablePropertySources propertySources) {

    }


    // ...省略部分代码

    // 委托给PropertySourcesPropertyResolver实现
    @Override
    @Nullable
    public String getProperty(String key) {
        return this.propertyResolver.getProperty(key);
    }

    // 委托给PropertySourcesPropertyResolver实现
    @Override
    public String resolvePlaceholders(String text) {
        return this.propertyResolver.resolvePlaceholders(text);
    }

    // ...省略部分代码
}
```

上面提到过，都是委托到`PropertySourcesPropertyResolver`，先看它的构造函数：

```java
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {

    @Nullable
    private final PropertySources propertySources;

    public PropertySourcesPropertyResolver(@Nullable PropertySources propertySources) {
        this.propertySources = propertySources;
    }

    ...省略代码
}
```

只依赖于一个`PropertySources`实例，这个`propertySources`就是`MutablePropertySources`的实例。重点分析一下```PropertySourcesPropertyResolver```类最复杂的一个方法：

```java
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {

    // ...省略代码

    /**
    * 根据指定的key获取属性值，如果获取到的属性值和targetValueType的类型不一致则进行类型转换，另外可以指定是否解析占位符
    *
    * 
    * @params key PropertySource中的属性key
    * @param targetValueType 属性值的目标类型
    * @param resolveNestedPlaceholders  是否解析占位符
    */
    @Nullable
    protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
        if (this.propertySources != null) {
            // 这里for循环的目的是从所有属性源中找出第一个存在指定key的属性，
            // 所以看到这里的代码应该知道MutablePropertySources类中的addFirst、addLast、addBefore等方法有什么作用了吧，
            // PropertySource的位置决定了在根据key获取属性时，如果多个属性源有相同的key值的属性时应该优先使用哪个属性源的属性。

            // 遍历所有的PropertySource属性源， 这里回看下MutablePropertySources的实现, 
            // 它实现了PropertySources接口，而PropertySources接口又继承了Iterable接口，所以是可迭代的
            for (PropertySource<?> propertySource : this.propertySources) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Searching for key '" + key + "' in PropertySource '" +
                                 propertySource.getName() + "'");
                }
                // 获取属性源中指定key的属性值
                Object value = propertySource.getProperty(key);
                //选用第一个不为null的匹配key的属性值
                if (value != null) {
                    // 判断是否处理解析嵌套的占位符，如${server.port}
                    if (resolveNestedPlaceholders && value instanceof String) {
                        // 处理属性占位符，如${server.port}，调用父类AbstractPropertyResolver的resolveNestedPlaceholders(String value)方法，
                        // 底层委托到PropertyPlaceholderHelper完成
                        value = resolveNestedPlaceholders((String) value);
                    }
                    logKeyFound(key, propertySource, value);
                    // 如果需要的话，进行一次类型转换，底层委托到DefaultConversionService完成
                    return convertValueIfNecessary(value, targetValueType);
                }
            }
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Could not find key '" + key + "' in any property source");
        }

        // 所有属性源中都不存在这个key
        return null;
    }

    // value转targetType类型
    @Nullable
    protected <T> T convertValueIfNecessary(Object value, @Nullable Class<T> targetType) {
        if (targetType == null) {
            return (T) value;
        }

        // 指定了targetType
        ConversionService conversionServiceToUse = this.conversionService;
        if (conversionServiceToUse == null) {
            // Avoid initialization of shared DefaultConversionService if
            // no standard type conversion is needed in the first place...
            // 判断value是否本身就是targetType类型的
            if (ClassUtils.isAssignableValue(targetType, value)) {
                return (T) value;
            }

            // 转换器为空，获取或者创建一个类型转换器， getSharedInstance是双重if判断实现的单例模式
            conversionServiceToUse = DefaultConversionService.getSharedInstance();
        }

        // 委托给DefaultConversionService进行具体的类型转换，将value转为targetType类型
        return conversionServiceToUse.convert(value, targetType);
    }

    ...省略代码
}
```

这里的源码告诉我们，如果出现多个`PropertySource`中存在同名的key，返回的是第一个`PropertySource`对应key的属性值的处理结果，因此我们如果需要自定义一些环境属性，需要十分清楚各个`PropertySource`的顺序。



# StandardEnvironment

表示“标准”(即非web)应用程序的Environment实现

```java
/**
* 适用于“标准”(即非web)应用程序的Environment实现。
* 除了ConfigurableEnvironment类的常用功能，如属性解析和与配置文件相关的操作外，当前类的实现还配置了两个默认的属性源，搜索顺序如下:
* AbstractEnvironment#getSystemProperties() 系统属性， 与jvm相关的， 比如java --server.port=8080 -jar xx.jar等针对每个jvm可以单独设置的
* AbstractEnvironment#getSystemEnvironment() 系统环境变量， 与操作系统相关的, 比如user.dir、path、JAVA_HOME等操作系统上配置的全局的
*
* 也就是说，如果名称为“xyz”的key同时存在于JVM系统属性及当前进程的环境变量集中，那么将返回来自系统属性System.getProperty(“xyz”)的值， 而不是System.getEnv("xyz")的值。
* 默认情况下使用这种顺序，因为系统属性是可以针对每个jvm单独设置的，而环境变量在给定系统上的多个jvm之间可能是相同的。赋予系统属性优先级允许在每个jvm的基础上重写环境变量。
* 这些默认属性源可以被删除、重新排序或替换掉;可以使用来自getPropertySources()的MutablePropertySources实例添加其他属性源。
* 有关使用示例，请参见ConfigurableEnvironment类的javadoc。
* 请参阅SystemEnvironmentPropertySource类的javadoc，了解关于在shell环境(例如Bash)中不允许变量名中包含周期字符的特殊属性名处理的详细信息。
*/
public class StandardEnvironment extends AbstractEnvironment {
    /** 系统环境变量的属性源名称， 作为PropertySource的name值 */
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

    /** JVM系统属性的属性源名称， 作为PropertySource的name值 */
    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

    /** 
    * 使用适合于任何Java标准环境的属性源定制MutablePropertySources:
    * systemEnvironment
	* systemProperties
	*
	* 存在于systemProperties中的属性优先级高于systemEnvironment中的属性，
	* 也就是说如果key为“xyz”的属性同时存在于systemProperties属性源和systemEnvironment属性源中,
	* 那么将返回systemProperties属性源中key为“xyz”的属性值（即返回jvm系统属性中的值而不是系统环境变量中的值）
	*
	* getSystemProperties()及getSystemEnvironment()方法是由父类AbstractEnvironment实现的
    */
    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        // 这里的属性源加入顺序决定了在调用MutablePropertySources.getProperty(String key)方法时优先返回哪个属性源的属性值

        // 将系统属性(System.getProperties()的返回值)作为key为systemProperties的属性源加入到MutablePropertySources中
        propertySources.addLast(
            new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));

        // 将系统环境变量(System.getEnv()的返回值)作为key为systemEnvironment的属性源加入到MutablePropertySources中
        propertySources.addLast(
            new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
    }
}
```

```getSystemProperties()```及```getSystemProperties()```方法的实现源码如下：

```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment { 
    // 获得JVM系统属性，与jvm相关的， 比如java --server.port=8080 -jar xx.jar等可以针对每个jvm单独设置的
    @Override
    @SuppressWarnings({"rawtypes", "unchecked"})
    public Map<String, Object> getSystemProperties() {
        try {

            return (Map) System.getProperties();
        }
        catch (AccessControlException ex) {
            return (Map) new ReadOnlySystemAttributesMap() {
                @Override
                @Nullable
                protected String getSystemAttribute(String attributeName) {
                    try {
                        return System.getProperty(attributeName);
                    }
                    catch (AccessControlException ex) {
                        if (logger.isInfoEnabled()) {
                            logger.info("Caught AccessControlException when accessing system property '" +
                                        attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
                        }
                        return null;
                    }
                }
            };
        }
    }

    // 获得系统环境变量,与操作系统相关的, 比如user.dir、path、JAVA_HOME等操作系统上配置的全局属性
    @Override
    @SuppressWarnings({"rawtypes", "unchecked"})
    public Map<String, Object> getSystemEnvironment() {
        if (suppressGetenvAccess()) {
            return Collections.emptyMap();
        }
        try {
            return (Map) System.getenv();
        }
        catch (AccessControlException ex) {
            return (Map) new ReadOnlySystemAttributesMap() {
                @Override
                @Nullable
                protected String getSystemAttribute(String attributeName) {
                    try {
                        return System.getenv(attributeName);
                    }
                    catch (AccessControlException ex) {
                        if (logger.isInfoEnabled()) {
                            logger.info("Caught AccessControlException when accessing system environment variable '" +
                                        attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
                        }
                        return null;
                    }
                }
            };
        }
    }
}
```



# StandardServletEnvironment

servlet环境的`Environment`实现

```java
/**
* 在基于Servlet的web应用中所使用的Environment实现，默认情况下，所有与web相关(基于servlet的)的ApplicationContext类都会初始化该类的一个实例。
* 
*/
public class StandardServletEnvironment extends StandardEnvironment implements ConfigurableWebEnvironment {

    /** Servlet context init parameters property source name: {@value}. */
    public static final String SERVLET_CONTEXT_PROPERTY_SOURCE_NAME = "servletContextInitParams";

    /** Servlet config init parameters property source name: {@value}. */
    public static final String SERVLET_CONFIG_PROPERTY_SOURCE_NAME = "servletConfigInitParams";

    /** JNDI property source name: {@value}. */
    public static final String JNDI_PROPERTY_SOURCE_NAME = "jndiProperties";

    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        // 添加基于ServletConfig的属性源到MutablePropertySources中， 属性源的key为servletConfigInitParams
        propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));

        // 添加基于ServletContext的属性源到MutablePropertySources中， 属性源的key为servletContextInitParams
        propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));

        // 默认的jndi环境是否可用
        if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
            propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
        }

        // 调用父类标准环境(StandardEnvironment)的customizePropertySources方法，添加系统属性和系统环境变量属性源到MutablePropertySources中
        super.customizePropertySources(propertySources);
    }

    @Override
    public void initPropertySources(@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
        // 添加ServletContextPropertySource及ServletConfigPropertySource属性源
        WebApplicationContextUtils.initServletPropertySources(getPropertySources(), servletContext, servletConfig);
    }
}
```

来看下WebApplicationContextUtils.initServletPropertySources方法的实现源码：

```java
/**
* 这是一个用于对给定的ServletContext对象获取WebApplicationContext的便捷的方法，
* 这个类的存在对于以编程方式从自定义web视图或MVC操作中访问Spring的应用程序上下文非常有用。
* 请注意，对于许多web框架(Spring家族中的一部分或作为外部库提供的框架)，有更方便的方法来访问根上下文。
* 这个工具类只是访问根上下文的最通用的方式。
*
* 参见org.springframework.web.context.ContextLoader
* 参见org.springframework.web.servlet.FrameworkServlet
* 参见org.springframework.web.servlet.DispatcherServlet
* 参见org.springframework.web.jsf.FacesContextUtils
* 参见org.springframework.web.jsf.el.SpringBeanFacesELResolver
*/
public abstract class WebApplicationContextUtils {
    @Nullable
    public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
        return getWebApplicationContext(sc, WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
    }

    @Nullable
    public static WebApplicationContext getWebApplicationContext(ServletContext sc, String attrName) {
        Assert.notNull(sc, "ServletContext must not be null");
        Object attr = sc.getAttribute(attrName);
        if (attr == null) {
            return null;
        }
        if (attr instanceof RuntimeException) {
            throw (RuntimeException) attr;
        }
        if (attr instanceof Error) {
            throw (Error) attr;
        }
        if (attr instanceof Exception) {
            throw new IllegalStateException((Exception) attr);
        }
        if (!(attr instanceof WebApplicationContext)) {
            throw new IllegalStateException("Context attribute is not of type WebApplicationContext: " + attr);
        }
        return (WebApplicationContext) attr;
    }

    // 添加ServletContextPropertySource及ServletConfigPropertySource属性源到environment(MutablePropertySources)中
    public static void initServletPropertySources(MutablePropertySources sources,
                                                  @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {

        Assert.notNull(sources, "'propertySources' must not be null");
        String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME;
        // 注意在StandardServletEnvironment#customizePropertySources(MutablePropertySources propertySources) 方法中已经将name为
        // "servletContextInitParams"及"servletConfigInitParams"的属性源添加到MutablePropertySources中了，只不过当时添加的属性源的类型为
        // StubPropertySource，这个类型的属性源代表一种需要延迟初始化（添加到环境中时还不确定具体的数据源是什么，只占个坑位）的属性源
        if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
            // 添加基于ServletContext的属性源ServletContextPropertySource
            sources.replace(name, new ServletContextPropertySource(name, servletContext));
        }
        name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME;
        if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
            // 添加基于ServletConfig的属性源ServletConfigPropertySource
            sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
        }
    }
}
```





# 总结

看到这里再回过头看Environment的类体系，想一想每个类的设计意图，做了哪些事情

![类的继承体系](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191231093757.png)

<img src="https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20191231100908.png" style="zoom: 80%;" />