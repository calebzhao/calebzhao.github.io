---
title: spring入门
date: 2020-01-03 20:20:24
tags:
- spring
categories:
- spring
---



# spring ioc

## 循环引用

造成循环引用的示例，有2个service类，名称分别为AService.java，BService.java

- AService.java

```java
package com.calebzhao.spring.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Service;

/**
 * @author calebzhao
 * @date 2020/1/3 20:10
 */
@Service
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class AService {

    @Autowired
    private BService bService;

    public void test(){
        bService.test();
        System.out.println("AService test");
    }
}
```

**注意AService声明了`@@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`**

- BService.java

```java
package com.calebzhao.spring.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Service;

/**
 * @author calebzhao
 * @date 2020/1/3 20:10
 */
@Service
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class BService {

    @Autowired
    private AService aService;

    public void test(){
        System.out.println("BService test");
    }
}

```

**注意BService声明了`@@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`**

- 入口类：

```java
package com.calebzhao.spring;

import com.calebzhao.spring.config.MyApplicationContextInitializer;
import com.calebzhao.spring.service.AService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

/**
 * @author calebzhao
 * @date 2020/1/3 20:07
 */
@SpringBootApplication
public class SpringDemoApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(SpringDemoApplication.class);

        ConfigurableApplicationContext applicationContext = application.run(args);

        AService aService = applicationContext.getBean(AService.class);
        aService.test();
    }
}
```

启动会报错：

```
"C:\Program Files\Java\jdk1.8.0_201\bin\java.exe" -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2019.3\lib\idea_rt.jar=56442:C:\Program Files\JetBrains\IntelliJ IDEA 2019.3\bin" -Dfile.encoding=UTF-8 -classpath "C:\Program Files\Java\jdk1.8.0_201\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\deploy.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\access-bridge-64.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\cldrdata.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\dnsns.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\jaccess.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\jfxrt.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\localedata.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\nashorn.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunec.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunjce_provider.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunmscapi.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunpkcs11.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\zipfs.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\javaws.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\jfxswt.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\management-agent.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\plugin.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_201\jre\lib\rt.jar;F:\code\spring-cloud-learn2\spring-demo\target\classes;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter-web\2.2.1.RELEASE\spring-boot-starter-web-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter\2.2.1.RELEASE\spring-boot-starter-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot\2.2.1.RELEASE\spring-boot-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-autoconfigure\2.2.1.RELEASE\spring-boot-autoconfigure-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter-logging\2.2.1.RELEASE\spring-boot-starter-logging-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\slf4j\slf4j-api\1.7.29\slf4j-api-1.7.29.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\apache\logging\log4j\log4j-to-slf4j\2.12.1\log4j-to-slf4j-2.12.1.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\apache\logging\log4j\log4j-api\2.12.1\log4j-api-2.12.1.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\slf4j\jul-to-slf4j\1.7.29\jul-to-slf4j-1.7.29.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-core\5.2.1.RELEASE\spring-core-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-jcl\5.2.1.RELEASE\spring-jcl-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\yaml\snakeyaml\1.25\snakeyaml-1.25.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter-json\2.2.1.RELEASE\spring-boot-starter-json-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\core\jackson-databind\2.10.0\jackson-databind-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\core\jackson-annotations\2.10.0\jackson-annotations-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\core\jackson-core\2.10.0\jackson-core-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.10.0\jackson-datatype-jdk8-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.10.0\jackson-datatype-jsr310-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\jackson\module\jackson-module-parameter-names\2.10.0\jackson-module-parameter-names-2.10.0.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter-tomcat\2.2.1.RELEASE\spring-boot-starter-tomcat-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\apache\tomcat\embed\tomcat-embed-core\9.0.27\tomcat-embed-core-9.0.27.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\apache\tomcat\embed\tomcat-embed-el\9.0.27\tomcat-embed-el-9.0.27.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.27\tomcat-embed-websocket-9.0.27.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\boot\spring-boot-starter-validation\2.2.1.RELEASE\spring-boot-starter-validation-2.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\jakarta\validation\jakarta.validation-api\2.0.1\jakarta.validation-api-2.0.1.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\hibernate\validator\hibernate-validator\6.0.18.Final\hibernate-validator-6.0.18.Final.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\jboss\logging\jboss-logging\3.4.1.Final\jboss-logging-3.4.1.Final.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\com\fasterxml\classmate\1.5.1\classmate-1.5.1.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-web\5.2.1.RELEASE\spring-web-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-beans\5.2.1.RELEASE\spring-beans-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-webmvc\5.2.1.RELEASE\spring-webmvc-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-aop\5.2.1.RELEASE\spring-aop-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-context\5.2.1.RELEASE\spring-context-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\springframework\spring-expression\5.2.1.RELEASE\spring-expression-5.2.1.RELEASE.jar;F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\projectlombok\lombok\1.18.10\lombok-1.18.10.jar" com.calebzhao.spring.SpringDemoApplication

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.1.RELEASE)

initialzer
2020-01-03 20:16:47.788  INFO 23148 --- [           main] c.c.spring.SpringDemoApplication         : Starting SpringDemoApplication on calebzhao with PID 23148 (F:\code\spring-cloud-learn2\spring-demo\target\classes started by calebzhao in F:\code\spring-cloud-learn2)
2020-01-03 20:16:47.790  INFO 23148 --- [           main] c.c.spring.SpringDemoApplication         : No active profile set, falling back to default profiles: default
2020-01-03 20:16:48.444  INFO 23148 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-03 20:16:48.450  INFO 23148 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-03 20:16:48.450  INFO 23148 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.27]
2020-01-03 20:16:48.507  INFO 23148 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-03 20:16:48.507  INFO 23148 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 682 ms
2020-01-03 20:16:48.613  INFO 23148 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-03 20:16:48.730  INFO 23148 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-03 20:16:48.732  INFO 23148 --- [           main] c.c.spring.SpringDemoApplication         : Started SpringDemoApplication in 1.176 seconds (JVM running for 1.867)
Exception in thread "main" org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'AService': Unsatisfied dependency expressed through field 'bService'; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'BService': Unsatisfied dependency expressed through field 'aService'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'AService': Requested bean is currently in creation: Is there an unresolvable circular reference?
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:639)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:116)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessProperties(AutowiredAnnotationBeanPostProcessor.java:397)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1429)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:594)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:341)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:227)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveNamedBean(DefaultListableBeanFactory.java:1155)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveBean(DefaultListableBeanFactory.java:416)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:349)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:342)
	at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1126)
	at com.calebzhao.spring.SpringDemoApplication.main(SpringDemoApplication.java:22)
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'BService': Unsatisfied dependency expressed through field 'aService'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'AService': Requested bean is currently in creation: Is there an unresolvable circular reference?
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:639)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:116)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessProperties(AutowiredAnnotationBeanPostProcessor.java:397)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1429)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:594)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:341)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1287)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1207)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:636)
	... 13 more
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'AService': Requested bean is currently in creation: Is there an unresolvable circular reference?
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:267)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1287)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1207)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:636)
	... 24 more
```

