---
title: Class对象的getXXXClass和getXXXName
date: 2019-12-29 16:51:55
tags: 
- spring
- spring boot
- spring cloud
categories: spring boot
---



# 前言

为什么要有这篇文章？

当我们分析在spring boot源码时，经常会看到SpringFactoriesLoader.loadFactoryNames(xxx.classs)返回了很多自动化配置类的名称，比如EnableAutoConfiguration=xxxx， 虽然可以看到EnableAutoConfiguration具体加载的配置有哪些，但是它的值到底是从哪个项目来的，自动化配置了哪个项目可能还是很模糊，找对应的配置可能还需要花费一定的时间。



# 自动化配置的地方

## BootstrapConfiguration

- ```spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories``` 共4个

```

# Bootstrap components
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration,\
org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration,\
org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
```



- ```spring-cloud-netflix-eureka-client-2.2.0.RELEASE.jar!\META-INF\spring.factories```共1个

  ```
  org.springframework.cloud.bootstrap.BootstrapConfiguration=\
  org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
  ```

  



## ApplicationListener

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories``` 共9个

```
## Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```



- ```spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories``` 共3个

```
## Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
org.springframework.cloud.bootstrap.LoggingSystemShutdownListener,\
org.springframework.cloud.context.restart.RestartListener
```



- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories``` 共1个

```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```





## ApplicationContextInitializer

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Application Context Initializers
  org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
  org.springframework.boot.context.ContextIdApplicationContextInitializer,\
  org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
  org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
  org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
  ```

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories``` 共2个

  ```
  # Initializers
  org.springframework.context.ApplicationContextInitializer=\
  org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
  org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
  ```

  

## EnableAutoConfiguration

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories``` 共2个

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration,\
...省略
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
```



- ```spring-cloud-commons-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  # AutoConfiguration
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.client.CommonsClientAutoConfiguration,\
  org.springframework.cloud.client.ReactiveCommonsClientAutoConfiguration,\
  org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.client.discovery.composite.reactive.ReactiveCompositeDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.client.discovery.noop.NoopDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.client.discovery.simple.SimpleDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.client.discovery.simple.reactive.SimpleReactiveDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.client.hypermedia.CloudHypermediaAutoConfiguration,\
  org.springframework.cloud.client.loadbalancer.AsyncLoadBalancerAutoConfiguration,\
  org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration,\
  org.springframework.cloud.client.loadbalancer.reactive.LoadBalancerBeanPostProcessorAutoConfiguration,\
  org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerClientAutoConfiguration,\
  org.springframework.cloud.client.loadbalancer.reactive.ReactiveLoadBalancerAutoConfiguration,\
  org.springframework.cloud.client.serviceregistry.ServiceRegistryAutoConfiguration,\
  org.springframework.cloud.commons.httpclient.HttpClientConfiguration,\
  org.springframework.cloud.commons.util.UtilAutoConfiguration,\
  org.springframework.cloud.configuration.CompatibilityVerifierAutoConfiguration,\
  org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration
  ```

- ```spring-cloud-context-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  # AutoConfiguration
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
  org.springframework.cloud.autoconfigure.LifecycleMvcEndpointAutoConfiguration,\
  org.springframework.cloud.autoconfigure.RefreshAutoConfiguration,\
  org.springframework.cloud.autoconfigure.RefreshEndpointAutoConfiguration,\
  org.springframework.cloud.autoconfigure.WritableEnvironmentEndpointAutoConfiguration
  ```

- ```spring-cloud-loadbalancer-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  # AutoConfiguration
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.loadbalancer.config.LoadBalancerAutoConfiguration,\
  org.springframework.cloud.loadbalancer.config.BlockingLoadBalancerClientAutoConfiguration,\
  org.springframework.cloud.loadbalancer.config.LoadBalancerCacheAutoConfiguration
  ```

  

- ```spring-cloud-openfeign-core-2.2.0.RELEASE.jar!\META-INF\spring.factories```

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
org.springframework.cloud.openfeign.hateoas.FeignHalAutoConfiguration,\
org.springframework.cloud.openfeign.FeignAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.loadbalancer.FeignLoadBalancerAutoConfiguration
```



- ```spring-cloud-netflix-ribbon-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
  ```

- ```spring-cloud-netflix-hystrix-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.hystrix.HystrixAutoConfiguration,\
  org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerAutoConfiguration,\
  org.springframework.cloud.netflix.hystrix.security.HystrixSecurityAutoConfiguration
  ```

  



## EnvironmentPostProcessor

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

```
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor
```



- ```spring-cloud-commons-2.2.0.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Environment Post Processors
  org.springframework.boot.env.EnvironmentPostProcessor=\
  org.springframework.cloud.client.HostInfoEnvironmentPostProcessor
  ```



## PropertySourceLoader

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # PropertySource Loaders
  org.springframework.boot.env.PropertySourceLoader=\
  org.springframework.boot.env.PropertiesPropertySourceLoader,\
  org.springframework.boot.env.YamlPropertySourceLoader
  ```

  



## SpringApplicationRunListener

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  
  # Run Listeners
  org.springframework.boot.SpringApplicationRunListener=\
  org.springframework.boot.context.event.EventPublishingRunListener
  ```

  



## EnableCircuitBreaker

- ```spring-cloud-netflix-hystrix-2.2.0.RELEASE.jar!\META-INF\spring.factories```

```
org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration
```



## BeanInfoFactory

- ```spring-beans-5.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  org.springframework.beans.BeanInfoFactory=org.springframework.beans.ExtendedBeanInfoFactory
  ```

  

## AutoConfigurationImportListener

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Auto Configuration Import Listeners
  org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
  org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener
  
  ```

  



## AutoConfigurationImportFilter

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Auto Configuration Import Filters
  org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
  org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
  org.springframework.boot.autoconfigure.condition.OnClassCondition,\
  org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
  ```

  

## SpringBootExceptionReporter

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Error Reporters
  org.springframework.boot.SpringBootExceptionReporter=\
  org.springframework.boot.diagnostics.FailureAnalyzers
  ```

  



## TemplateAvailabilityProvider

- ```spring-boot-autoconfigure-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # Template availability providers
  org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
  org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
  org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
  org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
  org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
  org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider
  ```

  

## FailureAnalysisReporter

- ```spring-boot-2.2.1.RELEASE.jar!\META-INF\spring.factories```

  ```
  # FailureAnalysisReporters
  org.springframework.boot.diagnostics.FailureAnalysisReporter=\
  org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter
  ```

  