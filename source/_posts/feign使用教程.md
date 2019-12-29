---
title: feign使用教程
date: 2019-12-29 16:37:44
tags: 
- spring cloud
- feign
categories: spring cloud
---

# 简介
```Feign```是一款java的Restful客户端组件，```Feign```使得 Java HTTP 客户端编写更方便。Feign 灵感来源于```Retrofit```, ```JAXRS-2.0```和```WebSocket```。```Feign``` 最初是为了降低统一绑定Denominator 到 HTTP API 的复杂度，不区分是否支持 ReSTfulness。

# 为什么选择Feign而不是其他
你可以使用 ```Jersey``` 和 ```CXF``` 这些来写一个 Rest 或 SOAP 服务的java客服端。你也可以直接使用 Apache HttpClient 来实现。但是 Feign 的目的是尽量的减少资源和代码来实现和 HTTP API 的连接。通过自定义的编码解码器以及错误处理，你可以编写任何基于文本的 HTTP API。

# Feign工作机制
Feign 通过注解注入一个模板化请求进行工作。只需在发送之前关闭它，参数就可以被直接的运用到模板中。然而这也限制了 Feign，只支持文本形式的API，它在响应请求等方面极大的简化了系统。同时，它也是十分容易进行单元测试的。

# 基本用法
基本的使用如下所示，一个对于canonical Retrofit sample的适配。
```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);

}

public static class Contributor {
  String login;
  int contributions;
}

public static class Issue {
  String title;
  String body;
  List<String> assignees;
  int milestone;
  List<String> labels;
}

public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
  
    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

# 注解
| Annotation                                 | Target        | Usage                                                        |
| ------------------------------------------ | ------------- | ------------------------------------------------------------ |
| @RequestLine                               | Method        | 为请求定义HttpMethod 以及UriTemplate，被{expression}包裹的表达式可以通过方法上的@Param注解解析为真实的值 |
| [@Param](https://my.oschina.net/u/2303379) | Parameter     | 定义一个模板变量的值，其值通过用于在解析模板表达式变量时使用，如果该注解在模板表达式中被使用到了，则不会作为form表单提交参数，否则作为表单提交参数 |
| @Headers                                   | Method， Type | 定义HeaderTemplate， 是定义HeaderTemplate的一种变种，使用@Param注解的值来解析@Headers注解中对应的表达式， 当该注解被使用在类上时，该模板将被应用于所有的方法上， 注解被用在方法上时，该模板只会在被该注解标识的方法上生效 |
| @QueryMap                                  | Parameter     | 定义了key-value键值对的Map对象或者 POJO，以扩展成查询字符串  |
| @HeaderMap                                 | Parameter     | 定义了key-value键值对的Map对象，以扩展到Http Header          |
| @Body                                      | Method        | 定义1个模板,该模板类似于UriTemplate 或HeaderTemplate，使用@Params注解的值来解析@Body注解中对应的的表达式 |
> **重写请求行**  
> 当Feign被创建时如果需要将一个请求发送到不同的主机，就需要提供它，或者你想为每个请求提供一个目标主机，包含一个 ```java.net.URI```参数，Feign 将使用这个值作为请求目标。
> ```java
> @RequestLine("POST /repos/{owner}/{repo}/issues")
> void createIssue(URI host, Issue issue, @Param("owner") String owner, @Param("repo") String repo);
> ```

# 模板和表达式(Template And Expression)
Feign 表达式由 [URI 模板 -RFC 6570](https://tools.ietf.org/html/rfc6570 "URI 模板 -RFC 6570")定义的简单字符串表达式(级别1)所表示。 表达式使用其相应的 带有```Param``` 注解的方法参数展开。

例子
```java
public interface GitHub {
  
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repository);
  
  class Contributor {
    String login;
    int contributions;
  }
}

public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
    
    /* The owner and repository parameters will be used to expand the owner and repo expressions
     * defined in the RequestLine.
     * 
     * the resulting uri will be https://api.github.com/repos/OpenFeign/feign/contributors
     */
    github.contributors("OpenFeign", "feign");
  }
}
```
表达式必须用大括号```{}```括起来，可以包含正则表达式模式，中间用冒号分隔```:``` 以限制解析值。 示例```owner```必须按字母顺序排列。 ```{ owner: [ a-zA-Z ] * }```

# 请求参数展开
```@RequesLline``` 和 ```@QueryMap``` 模板遵循一级模板的[URITemplate-RFC 6570](https://tools.ietf.org/html/rfc6570 "URITemplate-RFC 6570")规范，它指定了以下内容:
- 省略未解析的表达式
- 如果尚未通过```@Param```注解对所有文字和变量值进行编码或标记为已编码，则所有文字和变量值均经过pct编码。

# 未定义 vs 空值(Undefined vs Empty)
未定义表达式是指表达式的值为显式```null```或未提供任何值(```""```)的表达式。 根据[URITemplate-RFC 6570](https://tools.ietf.org/html/rfc6570 "URITemplate-RFC 6570")，可以为表达式提供一个空值。当 Feign 解析一个表达式时，它首先确定该值是否已定义，如果已定义，那么将保留该查询参数。 如果未定义，则删除该查询参数。 有关完整的细节，请参见下文。

- 空字符串(Empty String)
```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", "");
   this.demoClient.test(parameters);
}
```
结果
```
http://localhost:8080/test?param=
```
- 没有值(Missing)
```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   this.demoClient.test(parameters);
}
```
结果
```
http://localhost:8080/test
```
- 未定义的(Undefined)
```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", null);
   this.demoClient.test(parameters);
}
```
结果
```
http://localhost:8080/test
```
更多示例请参见[高级用法](https://github.com/OpenFeign/feign#advanced-usage "高级用法")。
>**斜杠(/)呢?**   
>默认情况下,@requestline 和@querymap 模板不编码斜杠 / 字符。 若要更改此行为，请将@requestline 上的 decodeSlash 属性设置为 false。  
>
>**加号(+)呢？**  
>根据 URI 规范，URI 的路径和查询片段中都允许使用 ```+``` 符号，但是查询上的符号处理可能不一致。 在一些遗留系统中，```+``` 等价于一个空格。 Feign 采用了现代系统的方式来处理，在现代系统中，一个 ```+``` 符号不应该表示一个空格，当在一个查询字符串中找到```+```时，它被显式地编码为``` %2B ```  
>如果您希望使用 ```+``` 作为空格，那么可以使用文本字符或直接将值编码为``` %2B ``` 

# 定制请求参数展开(Expander)
```@Param``` 注解有一个可选的属性扩展器，允许对单个参数的扩展进行完全控制。 ```expander``` 属性必须引用一个```Expander``` 接口的实现类:
```java
public interface Expander {
    String expand(Object value);
}
```
此方法的结果遵循上述相的规则。如果结果为 ```null``` 或空字符串(```""```)，则省略该值。 如果值没有经过 pct 编码，那么也会像上面的规则那样做。 更多示例请参见 [自定义```@Param```扩展](https://github.com/OpenFeign/feign#custom-param-expansion "自定义```@Param```扩展")。

# 请求头(Header)展开
```Header``` 和 ```HeaderMap``` 模板遵循与 [请求参数扩展](https://github.com/OpenFeign/feign#request-parameter-expansion "请求参数扩展") 相同的规则，但有以下修改:
- 省略未解析的表达式。 如果header的结果为空值，则删除整个header
- 未执行 pct 编码

有关示例，请参阅[Headers](https://github.com/OpenFeign/feign#headers "Headers")。
> #### 关于```@Param``` 参数及其名称的说明: 
> 所有名称相同的表达式，不管它们在@requestline、@querymap、@bodytemplate 或@headers 上的位置如何，都将解析为相同的值。 在下面的示例中，contentType 的值将用于解析头部和路径表达式:
> ```java
> public interface ContentService {
> @RequestLine("GET /api/documents/{contentType}")
> @Headers("Accept: {contentType}")
> String getDocumentByType(@Param("contentType") String type);
> }
> ```
> 在设计接口时要牢记这一点。

# 请求体(Body)展开
请求体(Body)模板遵循与[请求参数](https://github.com/OpenFeign/feign#request-parameter-expansion "请求参数")展开相同的规则，具有以下修改:
- 省略未解析的表达式
- 展开的值在放到请求正文之前不会通过Encoder 编码器传值
- 必须指定```Content-Type```请求头，实例请参阅 [请求体模板](https://github.com/OpenFeign/feign#body-templates "请求体模板")

# 自定义
Feign 有许多可以自定义的方面。举个简单的例子，你可以使用 ```Feign.builder()``` 来构造一个拥有你自己组件的API接口。如下:
```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}
...
// AccountDecoder 是自己实现的一个Decoder
Bank bank = Feign.builder().decoder(new AccountDecoder()).target(Bank.class, "https://api.examplebank.com");
```

# 多种接口
Feign可以提供多种API接口，这些接口都被定义为```Target<T> ```(默认的实现是 ```HardCodedTarget<T>)```, 它允许在执行请求前动态发现和装饰该请求。

举个例子，下面的这个模式允许使用当前url和身份验证token来装饰每个发往身份验证中心服务的请求。
```java
CloudDNS cloudDNS = Feign.builder().target(new CloudIdentityTarget<CloudDNS>(user, apiKey));
```
# 示例
```Feign``` 包含了[GitHub](https://github.com/OpenFeign/feign/blob/master/example-github "GitHub") 和 [Wikipedia](https://github.com/OpenFeign/feign/blob/master/example-wikipedia "Wikipedia") 客户端的实现样例.相似的项目也同样在实践中运用了Feign。尤其是它的示例后台程序[example daemon](https://github.com/Netflix/denominator/tree/master/example-daemon "example daemon")。

# Feign集成模块
Feign 可以和其他的开源工具集成工作。你可以将这些开源工具集成到 Feign 中来。目前已经有的一些模块如下:

# Gson
Gson 包含了一个编码器和一个解码器，这个可以被用于JSON格式的API。
添加 ```GsonEncoder``` 以及 ```GsonDecoder``` 到你的 ```Feign.Builder``` 中， 如下:
```java
GsonCodec codec = new GsonCodec();
GitHub github = Feign.builder()
                     .encoder(new GsonEncoder())
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```

# Jackson
Jackson 包含了一个编码器和一个解码器，这个可以被用于JSON格式的API。
添加 ```JacksonEncoder``` 以及 ```JacksonDecoder``` 到你的 Feign.Builder 中， 如下:
```java
GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-jackson</artifactId>
    <version>8.18.0</version>
</dependency>
```

# Sax
```SaxDecoder``` 用于解析XML,并兼容普通JVM和Android。下面是一个配置sax来解析响应的例子:
```java
api = Feign.builder()
           .decoder(SAXDecoder.builder()
                              .registerContentHandler(UserIdHandler.class)
                              .build())
           .target(Api.class, "https://apihost");
```
Maven依赖:
```xml
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-sax</artifactId>
    <version>8.18.0</version>
</dependency>
```
# JAXB
JAXB 包含了一个编码器和一个解码器，这个可以被用于XML格式的API。
添加 ```JAXBEncoder``` 以及 ```JAXBDecoder``` 到你的 ```Feign.Builder``` 中， 如下:
```java
api = Feign.builder()
           .encoder(new JAXBEncoder())
           .decoder(new JAXBDecoder())
           .target(Api.class, "https://apihost");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-jaxb</artifactId>
    <version>8.18.0</version>
</dependency>
```
# JAX-RS
JAXRSContract 使用 JAX-RS 规范重写覆盖了默认的注解处理。下面是一个使用 JAX-RS 的例子:
```java
interface GitHub {
  [@GET](https://my.oschina.net/get) @Path("/repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@PathParam("owner") String owner, @PathParam("repo") String repo);
}

GitHub github = Feign.builder()
                     .contract(new JAXRSContract())
                     .target(GitHub.class, "https://api.github.com");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-jaxrs</artifactId>
    <version>8.18.0</version>
</dependency>
```

# OkHttp
```OkHttpClient``` 使用 OkHttp 来发送 Feign 的请求，OkHttp 支持 SPDY (SPDY是Google开发的基于TCP的传输层协议，用以最小化网络延迟，提升网络速度，优化用户的网络使用体验),并有更好的控制http请求。

要让 Feign 使用 OkHttp ，你需要将 OkHttp 加入到你的环境变量中区，然后配置 ```Feign``` 使用 ```OkHttpClient```，如下:
```java
GitHub github = Feign.builder()
                     .client(new OkHttpClient())
                     .target(GitHub.class, "https://api.github.com");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-okhttp</artifactId>
    <version>8.18.0</version>
</dependency>
```

# Ribbon
```RibbonClient``` 重写了 Feign 客户端的对URL的处理，其添加了 智能路由以及一些其他由Ribbon提供的弹性功能。
集成Ribbon需要你将ribbon的客户端名称当做url的host部分来传递，如下：
```
// myAppProd是你的ribbon client name
MyService api = Feign.builder().client(RibbonClient.create()).target(MyService.class, "https://myAppProd");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-ribbon</artifactId>
    <version>8.18.0</version>
</dependency>
```
# Java 11及 Http2
Http2Client 将 Feign 的 http 请求定向到在Java11中新加入的实现了http/2的 [HTTP/2 Client](http://www.javamagazine.mozaicreader.com/JulyAug2017#&pageSet=39&page=0 "HTTP/2 Client")。

要在Feign中使用新的 HTTP/2客户端，请使用Java SDK 11。 然后配置 Feign 使用 ```Http2Client```:
```java
GitHub github = Feign.builder()
                     .client(new Http2Client())
                     .target(GitHub.class, "https://api.github.com");
```

# Hystrix
```HystrixFeign``` 配置了 Hystrix 提供的熔断机制。
要在 Feign 中使用 Hystrix ，你需要添加Hystrix模块到你的环境变量，然后使用 ```HystrixFeign``` 来构造你的API:
```
MyService api = HystrixFeign.builder().target(MyService.class, "https://myAppProd");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-hystrix</artifactId>
    <version>8.18.0</version>
</dependency>
```
# SOAP
[SOAP](https://github.com/OpenFeign/feign/blob/master/soap "Soap") 包含一个编码器和解码器，可以与 XML API 一起使用。

该模块通过 JAXB 和 SOAPMessage 增加了对 SOAP Body 对象的编码和解码支持。 它还通过将 SOAPFault 解码功能包装到原始的 javax.xml.ws.soap.SOAPFaultException 中来提供 SOAPFault 解码功能，因此只需捕获 SOAPFaultException 即可处理 SOAPFault。

向 Feign. Builder 中添加 SOAPEncoder 和 / 或 SOAPDecoder，如下所示:

```java
public class Example {
  public static void main(String[] args) {
    Api api = Feign.builder()
	     .encoder(new SOAPEncoder(jaxbFactory))
	     .decoder(new SOAPDecoder(jaxbFactory))
	     .errorDecoder(new SOAPErrorDecoder())
	     .target(MyApi.class, "http://api");
  }
}
```
注意: 如果返回 SOAP 错误，响应了http错误码(4xx，5xx，...)，您可能还需要添加 SOAPErrorDecoder

# SLF4J
[SLF4JModule](https://github.com/OpenFeign/feign/blob/master/slf4j "```SLF4JModule```") 允许你使用 [SLF4J](http://www.slf4j.org/ "SLF4J") 作为 Feign 的日志记录模块，这样你就可以轻松的使用 Logback, Log4J , 等来记录你的日志.

要在 Feign 中使用 SLF4J ，你需要添加SLF4J模块和对应的日志记录实现模块(比如Log4J)到你的环境变量，然后配置Feign使用Slf4jLogger :
```java
GitHub github = Feign.builder()
                     .logger(new Slf4jLogger())
                     .target(GitHub.class, "https://api.github.com");
```
Maven依赖:
```xml
<!-- https://mvnrepository.com/artifact/com.netflix.feign/feign-gson -->
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-slf4j</artifactId>
    <version>8.18.0</version>
</dependency>
```
# Decoders
```Feign.builder()``` 允许你自定义一些额外的配置，比如说如何解码一个响应。假如有接口方法返回的消息不是 Response, String, byte[] 或者 void 类型的，那么你需要配置一个非默认的解码器。
下面是一个配置使用JSON解码器(使用的是feign-gson扩展)的例子:
```java
GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
```
假如你想在将响应传递给解码器处理前做一些额外的处理，那么你可以使用 ```mapAndDecode``` 方法。一个用例就是使用jsonp服务的时候:
```java
JsonpApi jsonpApi = Feign.builder()
                         .mapAndDecode((response, type) -> jsopUnwrap(response, type), new GsonDecoder())
                         .target(JsonpApi.class, "https://some-jsonp-api.com");
```

# Encoders
发送一个Post请求最简单的方法就是传递一个```String``` 或者 ```byte[]``` 类型的参数了。你也许还需添加一个```Content-Type```请求头，如下:
```java
interface LoginClient {
  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  void login(String content);
}
...
client.login("{\"user_name\": \"denominator\", \"password\": \"secret\"}");
```
通过配置一个解码器，你可以发送一个安全类型的请求体，如下是一个使用```feign-gson``` 扩展的例子:
```java
static class Credentials {
  final String user_name;
  final String password;

  Credentials(String user_name, String password) {
    this.user_name = user_name;
    this.password = password;
  }
}

interface LoginClient {
  @RequestLine("POST /")
  void login(Credentials creds);
}
...
LoginClient client = Feign.builder()
                          .encoder(new GsonEncoder())
                          .target(LoginClient.class, "https://foo.com");

client.login(new Credentials("denominator", "secret"));
```
# [@Body](https://my.oschina.net/u/3054742) templates
```@Body```注解申明一个请求体模板，模板中可以带有参数，与方法中 ```@Param``` 注解申明的参数相匹配,使用方法如下
```java
interface LoginClient {

  @RequestLine("POST /")
  @Headers("Content-Type: application/xml")
  @Body("<login \"user_name\"=\"{user_name}\" \"password\"=\"{password}\"/>")
  void xml(@Param("user_name") String user, @Param("password") String password);

  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  // json curly braces must be escaped!
  // 这里JSON格式需要的花括号居然需要转码，有点蛋疼了。
  @Body("%7B\"user_name\": \"{user_name}\", \"password\": \"{password}\"%7D")
  void json(@Param("user_name") String user, @Param("password") String password);
}
...
// <login "user_name"="denominator" "password"="secret"/>
client.xml("denominator", "secret");
// {"user_name": "denominator", "password": "secret"}
client.json("denominator", "secret");
```

# Headers
Feign 支持给请求的api设置或者请求的客户端设置请求头

使用 ```@Headers``` 设置静态请求头
```
// 给BaseApi中的所有方法设置Accept请求头
@Headers("Accept: application/json")
interface BaseApi<V> {
  // 单独给put方法设置Content-Type请求头
  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String, V value);
}
```
# 设置动态值的请求头
```
@RequestLine("POST /")
@Headers("X-Ping: {token}")
void post(@Param("token") String token);
```
设置key和value都是动态的请求头
有些API需要根据调用时动态确定使用不同的请求头(e.g. custom metadata header fields such as “x-amz-meta-” or “x-goog-meta-“),这时候可以使用 ```@HeaderMap```注解，如下:
```java
// @HeaderMap 注解设置的请求头优先于其他方式设置的
@RequestLine("POST /")
void post(@HeaderMap Map<String, Object> headerMap);
```
# 给Target设置请求头(Target)
有时我们需要在一个API实现中根据不同的endpoint来传入不同的Header，这个时候我们可以使用自定义的```RequestInterceptor``` 或 ```Target```来实现.
通过自定义的 ```RequestInterceptor``` 来实现请查看 Request Interceptors 章节.
下面是一个通过自定义Target来实现给每个Target设置安全校验信息Header的例子:
```
static class DynamicAuthTokenTarget<T> implements Target<T> {
  public DynamicAuthTokenTarget(Class<T> clazz,
                                UrlAndTokenProvider provider,
                                ThreadLocal<String> requestIdProvider);
  ...
  @Override
  public Request apply(RequestTemplate input) {
    TokenIdAndPublicURL urlAndToken = provider.get();
    if (input.url().indexOf("http") != 0) {
      input.insert(0, urlAndToken.publicURL);
    }
    input.header("X-Auth-Token", urlAndToken.tokenId);
    input.header("X-Request-ID", requestIdProvider.get());

    return input.request();
  }
}
...
Bank bank = Feign.builder()
        .target(new DynamicAuthTokenTarget(Bank.class, provider, requestIdProvider));
```
这种方法的实现依赖于给Feign 客户端设置的自定义的```RequestInterceptor``` 或 ```Target```。可以被用来给一个客户端的所有api请求设置请求头。比如说可是被用来在header中设置身份校验信息。这些方法是在线程执行api请求的时候才会执行，所以是允许在运行时根据上下文来动态设置header的。
比如说可以根据线程本地存储(thread-local storage)来为不同的线程设置不同的请求头。

# 高级用法
## Base APIS
有些请求中的一些方法是通用的，但是可能会有不同的参数类型或者返回类型，这个时候可以这么用:
```java
// 通用API
interface BaseAPI {
  @RequestLine("GET /health")
  String health();

  @RequestLine("GET /all")
  List<Entity> all();
}
```

## 继承通用API
```java
interface CustomAPI extends BaseAPI {
  @RequestLine("GET /custom")
  String custom();
}
```

## 各种类型有相同的表现形式，定义一个统一的API
```java
@Headers("Accept: application/json")
interface BaseApi<V> {

  @RequestLine("GET /api/{key}")
  V get(@Param("key") String key);

  @RequestLine("GET /api")
  List<V> list();

  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}

// 根据不同的类型来继承
interface FooApi extends BaseApi<Foo> { }

interface BarApi extends BaseApi<Bar> { }
```

# 日志(Logger)
你可以通过设置一个 Logger 来记录http消息，如下:
```
GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .logger(new Logger.JavaLogger().appendToFile("logs/http.log"))
                     .logLevel(Logger.Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
```
也可以参考上面的 SLF4J 章节的说明

# 请求拦截器(RequestInterceptor )
当你希望修改所有的的请求的时候，你可以使用Request Interceptors。比如说，你作为一个中介，你可能需要为每个请求设置 X-Forwarded-For
```
static class ForwardedForInterceptor implements RequestInterceptor {
  @Override public void apply(RequestTemplate template) {
    template.header("X-Forwarded-For", "origin.host.com");
  }
}

public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new ForwardedForInterceptor())
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```
拦截器的另一个常见示例是身份验证，这里有一个内置的基础校验拦截器 BasicAuthRequestInterceptor
```java
public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new BasicAuthRequestInterceptor(username, password))
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```
# 自定义 [@Param](https://my.oschina.net/u/2303379) 展开
在使用 ```@Param``` 注解给模板中的参数设值的时候，默认的是使用的对象的 ```toString()``` 方法的值，通过声明 自定义的```Param.Expander```，用户可以控制其行为，比如说格式化 ```Date``` 类型的值:
```
// 通过设置 @Param 的 expander 为 DateToMillis.class 可以定义Date类型的值
@RequestLine("GET /?since={date}")
Result list(@Param(value = "date", expander = DateToMillis.class) Date date);
```

# 动态查询参数(@QueryMap)
动态查询参数支持，通过使用 ```@QueryMap``` 可以允许动态传入请求参数,如下:
```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap Map<String, Object> queryMap);
}
```

这也可用于使用 ```QueryMapEncoder``` 从 POJO 对象生成查询参数。
```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap CustomPojo customPojo);
}
```
以这种方式使用时，不需要指定自定义 ```QueryMapEncoder```，就可以使用成员变量名作为查询参数名生成查询映射。 下面的 POJO 将生成```" / find?name={name}&number={number}"```的查询参数(不保证包含查询参数的顺序，并且通常情况下，如果任何值为 null，都会忽略它)。

设置一个自定义的 ```QueryMapEncoder```:
```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new MyCustomQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

当在对象上使用@QueryMap 注解时，默认的编码器会使用反射来检查对象提供的字段，以将对象的值扩展为查询字符串。 如果您希望使用 getter 和 setter 方法构建查询字符串，类似于JavaBean API 中定义的那样，请使用 ```BeanQueryMapEncoder```

# 错误处理(Error Handling)
如果需要对处理意外响应进行更多控制，那么 Feign 实例可以通过构建器注册自定义的 ```ErrorDecoder```。
```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .errorDecoder(new MyErrorDecoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```
所有导致 HTTP 状态不在```2xx``` 范围内的响应都将触发 ```ErrorDecoder``` 的 ```decode``` 方法，允许您处理响应、将故障包装成自定义异常或执行任何其他处理。 如果要重试请求，请抛出 ```RetryableException```。 这将调用已注册的 ```Retryer```。
如果有需要，Feign允许您为每个请求单独维护重试状态，那么每个Client在执行时都将创建一个```Retryer```实例。

如果重试确定不成功，最后将抛出```RetryException```。 若要抛出导致重试失败的具体原因，请使用 ```exceptionPropagationPolicy ()```选项来创建您的 Feign 客户端。

# 重试(Retry)
默认情况下，Feign会自动重试```IOException```，而与HTTP方法无关，将其视为与网络临时相关的异常，以及从```ErrorDecoder```抛出的任何```RetryableException```。 要定制此行为，请通过构建器注册定制的Retryer实例。
```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .retryer(new MyRetryer())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```
Retryer 负责根据```continueOrPropagate (RetryableException e)```方法中返回 的 true 或 false 来确定是否应该进行重试

# 静态方法及默认方法（static method and default method）
如果你使用的是JDK 1.8+ 的话，那么你可以给接口设置统一的默认方法和静态方法，这是JDK8的新特性，示例如下:
```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("GET /users/{username}/repos?sort={sort}")
  List<Repo> repos(@Param("username") String owner, @Param("sort") String sort);

  default List<Repo> repos(String owner) {
    return repos(owner, "full_name");
  }

  /**
   * Lists all contributors for all repos owned by a user.
   */
  default List<Contributor> contributors(String user) {
    MergingContributorList contributors = new MergingContributorList();
    for(Repo repo : this.repos(owner)) {
      contributors.addAll(this.contributors(user, repo.getName()));
    }
    return contributors.mergeResult();
  }

  static GitHub connect() {
    return Feign.builder()
                .decoder(new GsonDecoder())
                .target(GitHub.class, "https://api.github.com");
  }
}
```
英文原文请参考:https://github.com/OpenFeign/feign  
翻译参考:https://blog.csdn.net/u010862794/article/details/73649616