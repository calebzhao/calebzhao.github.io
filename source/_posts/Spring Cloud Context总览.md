---
layout: spring
title: Context总览
date: 2019-12-29 16:46:33
tags: spring cloud
categories: spring cloud
---



spring cloud 中的许多特性都已经被spring boot实现，其他一些特性主要是通过spring cloud context 和 spring cloud commons来实现的。spring cloud context提供了一些常用的公共类以及spring cloud 特有的服务(bootstrap context, encryption, refresh scope, and environment endpoints)；spring cloud commons提供了一系列的抽象类和公共类，供spring cloud不同的实现组件去使用(such as Spring Cloud Netflix and Spring Cloud Consul).。

# 1. Spring Cloud Context: Application Context Services
通过spring boot 构建 spring application已经很方便了，她还提供了一些监控、管理相关的endpoint; 由于 spring cloud 是建立在spring boot基础之上的，所以spring boot 的功能spring cloud都满足，此外还提供了一些特有的特性，这些特性可能在所有的spring cloud组件中都会（或者偶尔）使用到

## 1.1 The Bootstrap Application Context
spring cloud 应用中创建一个 bootstrap context，她是作为了主应用的 parent context ；主要负责从外部资源（主要指配置中心）加载配置以及在解码本地的配置文件。appliction context 和 bootstrap context 共享了相同Environment. 默认情况下，bootstrap properties(从配置中心加载的属性) 有更高的优先级，所以它们不能被本地的配置所覆盖。

bootstrap context使用了和主应用的 context 不同的配置方式，使用的是 bootstrap.yml 而不是 ```application.yml```，这主要是为了保证两个context的配置可以很好的分离；下面是 ```bootstrap.yml``` 的例子：
```yml
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```
如果需要禁用 bootstrap context ，我们可以设置环境变量```spring.cloud.bootstrap.enabled=false```



## 1.2 Application Context Hierarchies
如果你通过SpringApplication 或者 ```SpringApplicationBuilder```来构建你的应用，那么Bootstrap context 将会作为application context 的 parent context。子容器会继承从父容器中的属性，新增的属性有：


- bootstrap : 如果在Bootstrap context有任何非空的属性,那么这些属性的优先级都更高，比如说从 spring cloud config 加载的配置，默认优先级就比本地配置高；后面会提到如何覆盖这些属性

- applicationConfig : 如果你配置bootstrap.yml, 那么bootstrap.yml中的属性会用来配置 Bootstrap context；当Bootstrap context配置完成后就会把这些属性添加到application context 中，这些属性的优先级会比配置在application.yml 中的低
由于bootstrap.yml 的优先级比较低，所以可以用来设置一些属性的默认值

spring cloud 允许你扩展ApplicationContext ,例如可以使用 ApplicationContext已有的接口或者使用```SpringApplicationBuilder```提供的方法(parent(), child() and sibling())；这个bootstrap context是所有容器的父容器

## 1.3 Changing the Location of Bootstrap Properties
能够通过设置环境变量```spring.cloud.bootstrap.name```（默认是bootstrap）配置引导文件（```bootstrap.yml```）的名字，通过 ```spring.cloud.bootstrap.location```(默认是空)配置路径。
例如在环境变量中配置：
```
spring.cloud.bootstrap.name=start
spring.cloud.bootstrap.location=classpath:/test/
```
以上配置表明会加载```classpath:/test/start.yml(start.properties)```

## 1.4 Overriding the Values of Remote Properties
通过bootstrap context从远程添加的属性默认情况下是不能被本地配置所覆盖的。如果你想要使用系统变量或者配置文件来覆盖远程的属性，在**远程配置中设置**属性```spring.cloud.config.allowOverride=true```(注意：**在本地文件中设置是无效的**)来授权，同时需要配置哪些可以覆盖：
```
#远程属性不覆盖任何本地属性（本地任何属性都可以覆盖远程属性）
spring.cloud.config.overrideNone=true 

#远程属性不覆盖任何本地系统属性（本地环境变量，命令行参数可以覆盖远程属性，本地配置文件属性不能覆盖远程属性）
spring.cloud.config.overrideSystemProperties=false 
```
## 1.5 Customizing the Bootstrap Configuration
bootstrap context通过```/META-INF/spring.factories```配置的类初始化的所有的Bean都会在```SpingApplicatin```启动前加入到它的上下文里去。```spring.factories```中key 使用 ```org.springframework.cloud.bootstrap.BootstrapConfiguration```, 所有需要用来配置bootstrap context的配置类用逗号分隔；可以通过```@Order```来更改初始化序列，默认是”last”

spring-cloud-context.jar包的```/META-INF/spring.factories```源码如下：
```
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


首先使用```spring.factories```中配置的类来创建bootstrap context，然后所有使用@Bean注解的```ApplicationContextInitializer```将会被添加 main application context

## 1.6 Customizing the Bootstrap Property Sources
默认情况下外部化配置都是从配置中心获取属性，但是也可以通过实现```PropertySourceLocator```来实现添加属性到bootstrap context（需要把实现类添加到spring.factories）；比如可以从不同的数据库或者服务器获取配置。
```
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```

如果你把这个类打包到了一个jar里面，添加下面的配置到META-INF/spring.factories里面，那么所有依赖了这个jar的都会有customProperty这个配置
```
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
```
## 1.8 Environment Changes
应用可以监听```EnvironmentChangeEvent```来响应环境变量被改变的事件。一旦```EnvironmentChangeEvent```事件被触发，应用程序将会做这些事情：

- 重新绑定使用了```@ConfigurationProperties```的类
- 根据```logging.level.*```来设置应用的日志级别
默认情况下，Config Client不轮询Environment的改变。一般情况，不建议使用这种方式来监测变化（虽然你可以通过@Scheduled注解来设置）。对于可扩展的应用程序，使用广播```EnvironmentChangeEvent```到所有实例的方式，好过轮询的方式。（比如使用Spring Cloud Bus项目）。

## 1.9 Refresh Scope
当有配置发生变化的时候，使用了```@RefreshScope```的类将会被处理；这个特性解决了有状态bean只能在初始化注入配置的问题。例如：比如，在使用DataSource获得一个数据库连接的是时候，当通过Environment修改数据库连接字符串的时候，我们可以通过执行```@RefreshScope```来根据修改的配置获取一个新的URL的连接。
Refresh scope beans是延迟初始化的，在第一次使用的时候才初始化，这个scope充当了初始值的缓存；为了在下一次调用时强制初始化，必须使缓存无效。
RefreshScope在容器中是一个bean, 提供了一个public方法refreshAll()，通过清理当前的缓存来刷新，可以通过访问/refresh来触发刷新；也可以使用bean的名字来刷新refresh(String)，要使用还需要暴露出这个endpoint
```
management:
  endpoints:
    web:
      exposure:
        include: refresh
```
注意：```@RefreshScope```注解在一个```@Configuration```类上面，在重新初始化的时候需要注意到可以相互依赖造成的冲突。

# 1.11 Endpoints
相对于Spring Boot Actuator的应用，添加了一些管理端点：
```
POST to /actuator/env 更新Environment重新加载@ConfigurationProperties和日志级别
/actuator/refresh 重新初始化添加了@RefreshScope 的bean
/actuator/restart 重启初始化ApplicationContext，重启 (默认是禁用的)
/actuator/pause 调用ApplicationContext生命周期的方法stop()和start()
```
如果禁用了/actuator/restart，那么/actuator/pause和/actuator/resume都会被禁用

>  作者：9527华安  
>   链接：https://www.jianshu.com/p/a8b255c9feae  
>  来源：简书 
>  著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。