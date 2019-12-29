---
title: Feign源码学习
date: 2019-12-29 16:37:10
tags: 
- feign
- spring cloud
categories: spring cloud
---

# feign介绍
[Feign](https://github.com/OpenFeign/feign "`Feign")是一款java的Restful客户端组件，Feign使得 Java HTTP 客户端编写更方便。Feign 灵感来源于Retrofit, JAXRS-2.0和WebSocket。feign在github上有近5K个star，是一款相当优秀的开源组件，虽然相比Retrofit的近30K个star，逊色了太多，但是spring cloud集成了feign，使得feign在java生态中比Retrofit使用的更加广泛。

feign的基本原理是在接口方法上加注解，定义rest请求，构造出接口的动态代理对象，然后通过调用接口方法就可以发送http请求，并且自动解析http响应为方法返回值，极大的简化了客户端调用rest api的代码。官网的示例如下：
```
 interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
}

static class Contributor {
  String login;
  int contributions;
}

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
```
feign使用教程请参考官网https://github.com/OpenFeign/feign/

本文主要是对feign源码进行分析，根据源码来理解feign的设计架构和内部实现技术。

# Feign.build构建接口动态代理
我们先来看看接口的动态代理是如何构建出来的，下图是主要接口和类的类图：
![](https://oscimg.oschina.net/oscnet/up-0fe8e9d3c45abafd2164a3478432c5cb3f0.png)





























从上文中的示例可以看到，构建的接口动态代理对象是通过Feign.builder()生成Feign.Builder的构造者对象，然后设置相关的参数，再调用target方法构造的。Feign.Builder的参数包括：
```java
//拦截器，组装完RequestTemplate，发请求之前的拦截处理RequestTemplate
private final List<RequestInterceptor> requestInterceptors = new ArrayList<RequestInterceptor>();

//日志级别
private Logger.Level logLevel = Logger.Level.NONE;

//契约模型，默认为Contract.Default，用户创建MethodMetadata，用spring cloud就是扩展这个实现springMVC注解
private Contract contract = new Contract.Default();

//客户端，默认为Client.Default，可以扩展ApacheHttpClient，OKHttpClient，RibbonClient等
private Client client = new Client.Default(null, null);

//重试设置，默认不设置
private Retryer retryer = new Retryer.Default();

//日志，可以接入Slf4j
private Logger logger = new NoOpLogger();

//编码器，用于body的编码
private Encoder encoder = new Encoder.Default();

//解码器，用户response的解码
private Decoder decoder = new Decoder.Default();

//用@QueryMap注解的参数编码器
private QueryMapEncoder queryMapEncoder = new QueryMapEncoder.Default();

//请求错误解码器
private ErrorDecoder errorDecoder = new ErrorDecoder.Default();

//参数配置，主要是超时时间之类的
private Options options = new Options();

//动态代理工厂
private InvocationHandlerFactory invocationHandlerFactory = new InvocationHandlerFactory.Default();

//是否decode404
private boolean decode404;

private boolean closeAfterDecode = true;

// 异常隔离策略
private ExceptionPropagationPolicy propagationPolicy = NONE;
```

这块是一个典型的构造者模式，```target```方法内部先调用```build```方法新建一个```ReflectFeign```对象，然后调用```ReflectFeign```的```newInstance```方法创建动态代理，代码如下：
```java
//默认使用HardCodedTarget
public <T> T target(Class<T> apiType, String url) {
	return target(new HardCodedTarget<T>(apiType, url));
}

public <T> T target(Target<T> target) {
	return build().newInstance(target);
}

public Feign build() {
	SynchronousMethodHandler.Factory synchronousMethodHandlerFactory = new SynchronousMethodHandler.Factory(
								client, 
								retryer,
								requestInterceptors, 
								logger,
								logLevel, 
								decode404, 
								closeAfterDecode);
	
	// 该类中持有的contract非常重要（对接口及接口的方法上的注解解析）
	ParseHandlersByName handlersByName = new ParseHandlersByName(contract, 
								options, 
								encoder, 
								decoder, 
								queryMapEncoder,
								errorDecoder, 
								synchronousMethodHandlerFactory);
	
	// handlersByName将所有参数进行封装，并提供解析接口方法的逻辑
	// invocationHandlerFactory是Builder的属性，默认值是InvocationHandlerFactory.Default
	// 用创建java动态代理的InvocationHandler实现
	return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

```ReflectiveFeign```构造函数有三个参数：
- ```ParseHandlersByName``` 将builder所有参数进行封装，并提供解析接口方法的逻辑
- ```InvocationHandlerFactory``` java动态代理的InvocationHandler的工厂类，默认值是```InvocationHandlerFactory.Default```
- ```QueryMapEncoder``` 接口参数注解```@QueryMap```时，参数的编码器

```ReflectiveFeign.newInstance```方法创建接口动态代理对象：
```java
public <T> T newInstance(Target<T> target) {
	//targetToHandlersByName是构造器传入的ParseHandlersByName对象，根据target对象生成MethodHandler映射
	// 这里的MethodHandler的具体类型是在targetToHandlersByName.apply方法中使用工厂
	// SynchronousMethodHandler.Factory创建的SynchronousMethodHandler
	Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
	Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
	List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
	//遍历接口所有方法，构建Method->MethodHandler的映射
	for (Method method : target.type().getMethods()) {
		if (method.getDeclaringClass() == Object.class) {
			// Object类中的方法跳过
			continue;
		} else if(Util.isDefault(method)) {
			//接口default方法的Handler，这类方法直接调用
			DefaultMethodHandler handler = new DefaultMethodHandler(method);
			defaultMethodHandlers.add(handler);
			methodToHandler.put(method, handler);
		} else {
			methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
		}
	}
	//这里factory是构造其中传入的，创建InvocationHandler
	InvocationHandler handler = factory.create(target, methodToHandler);
	//java的动态代理
	T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);
	//将default方法直接绑定到动态代理上
	for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
		// 用到了java7的MethodHandler、Lookup
		defaultMethodHandler.bindTo(proxy);
	}
	return proxy;
}
```
这段代码主要的逻辑是：

- 创建MethodHandler的映射，这里创建的是实现类SynchronousMethodHandler
- 通过InvocationHandlerFatory创建InvocationHandler
- 绑定接口的default方法，通过DefaultMethodHandler绑定

类图中已经画出，```SynchronousMethodHandler```和```DefaultMethodHandler```实现了```InvocationHandlerFactory.MethodHandler```接口，动态代理对象调用方法时，如果是default方法，会直接调用接口方法，因为这里将接口的default方法绑定到动态代理对象上了，其他方法根据方法签名找到```SynchronousMethodHandler```对象，调用其invoke方法。

```Feign.configKey(Class targetType, Method method)```的实现源码：
```java
/**
* 例如User类中有 void list(Map map，int pageSize)将会生成User#list(User,int)签名
* 例如User类中有 void list()将会生成User#list()签名
*/
public static String configKey(Class targetType, Method method) {
	StringBuilder builder = new StringBuilder();
	// 拼接类名
	builder.append(targetType.getSimpleName());
	// 拼接#(
	builder.append('#').append(method.getName()).append('(');
	for (Type param : method.getGenericParameterTypes()) {
		param = Types.resolve(targetType, targetType, param);
		// 拼接参数类型及1个逗号，例如int,
		builder.append(Types.getRawType(param).getSimpleName()).append(',');
	}
	// 删除最后1个逗号
	if (method.getParameterTypes().length > 0) {
		builder.deleteCharAt(builder.length() - 1);
	}
	//拼接反括号)
	return builder.append(')').toString();
}
```

# Target
```Target```源码如下：
```java
public interface Target<T> {
	// api 接口类
	Class<T> type();

	// 名称
	String name();

	// api接口路径的前缀地址(例如http://localhost:8080)， 发送请求时会用这个地址和@RequestLine中的url路径（/user/list）
	// 拼接在一起作为请求url(http://localhost:8080/user/list)
	String url();

	public Request apply(RequestTemplate input);

	/**
	* Target的默认实现，什么也不做
	*/
	public static class HardCodedTarget<T> implements Target<T> {

		private final Class<T> type;
		private final String name;
		private final String url;

		public HardCodedTarget(Class<T> type, String url) {
			// 可以看到这里name默认和url一样
			this(type, url, url);
		}

		public HardCodedTarget(Class<T> type, String name, String url) {
			this.type = checkNotNull(type, "type");
			this.name = checkNotNull(emptyToNull(name), "name");
			this.url = checkNotNull(emptyToNull(url), "url");
		}
		
		@Override
		public Request apply(RequestTemplate input) {
			if (input.url().indexOf("http") != 0) {
				input.target(url());
			}
			return input.request();
		}
	}
}
```


# 创建MethodHandler方法处理器
SynchronousMethodHandler是feign组件的核心，接口方法调用转换为http请求和解析http响应都是通过SynchronousMethodHandler来执行的，相关类图如下：
![](https://oscimg.oschina.net/oscnet/up-668e4a442da56cb48fa3b4cb3bf4fb2ec15.png)

创建MethodHandler实现类```ParseHandlersByName```的代码：
```java
static final class ParseHandlersByName {
	private final Contract contract;
	private final Options options;
	private final Encoder encoder;
	private final Decoder decoder;
	private final ErrorDecoder errorDecoder;
	private final QueryMapEncoder queryMapEncoder;
	private final SynchronousMethodHandler.Factory factory;

	ParseHandlersByName(
		Contract contract,
		Options options,
		Encoder encoder,
		Decoder decoder,
		QueryMapEncoder queryMapEncoder,
		ErrorDecoder errorDecoder,
		SynchronousMethodHandler.Factory factory) {
		...省略
	}

	public Map<String, MethodHandler> apply(Target key) {
		 //通过contract解析接口方法，生成MethodMetadata列表，默认的contract解析Feign自定义的http注解
		List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
		
		Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
		for (MethodMetadata md : metadata) {
			//BuildTemplateByResolvingArgs实现RequestTemplate.Factory，RequestTemplate的工厂
			BuildTemplateByResolvingArgs buildTemplate;
			//根据方法元数据，使用不同的RequestTemplate的工厂
			if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
				 //如果有formParam，并且bodyTemplate不为空，请求体为x-www-form-urlencoded格式
				//将会解析form参数，填充到bodyTemplate中
				buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
			} 
			else if (md.bodyIndex() != null) {
				 //如果包含请求体，将会用encoder编码请求体对象
				buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
			}
			else {
				 //没有请求体，不需要编码器，使用默认的RequestTemplate的工厂
				buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
			}
			
			 //使用工厂SynchronousMethodHandler.Factory创建SynchronousMethodHandler
			result.put(md.configKey(),
					   factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
		}
		return result;
	}
}
```
这段代码的逻辑是：

1. 通过```Contract```解析接口方法，生成```MethodMetadata```，默认的```Contract```解析```Feign```自定义的http注解
2. 根据```MethodMetadata```方法元数据生成特定的```RequestTemplate```的工厂
3. 使用```SynchronousMethodHandler.Factory```工厂创建```SynchronousMethodHandler```

这里有两个工厂不要搞混淆了，```SynchronousMethodHandler```工厂和```RequestTemplate```工厂，```SynchronousMethodHandler```的属性包含```RequestTemplate```工厂

# Contract解析接口方法生成MethodMetadata
feign默认的解析器是```Contract.Default```继承了```Contract.BaseContract```，解析生成```MethodMetadata```方法入口：
```java
public interface Contract {
	/**
	* 解析feign api 类上的所有方法元数据（feign默认是解析其自定义的注解，
	* spring cloud openfeign继承了feign的默认契约，同时支持spring mvc自己的@Requestmapping主键）
	*/
	List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType);
	
	/**
	* 使用注解实现契约的通用逻辑， SpringMvcContract继承了BaseContract
	*/
	abstract class BaseContract implements Contract {
		/**
		* 解析targetType类中的所有方法
		* @param targetType 我们自己编写的feign api接口（标有@FeignClient注解的类）
		*/
		@Override
		public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
			// api接口不能有泛型
			checkState(targetType.getTypeParameters().length == 0, "Parameterized types unsupported: %s",
					   targetType.getSimpleName());
			// 最多只能有1个父接口，多余1个父接口则抛出异常（不能多继承父接口，只允许单继承接口）
			checkState(targetType.getInterfaces().length <= 1, "Only single inheritance supported: %s",
					   targetType.getSimpleName());
			// 只有1个父接口
			if (targetType.getInterfaces().length == 1) 
				// 检查父接口是否还有父接口，如果有则抛出异常（只允许1级父接口）
				checkState(targetType.getInterfaces()[0].getInterfaces().length == 0,
						   "Only single-level inheritance supported: %s",
						   targetType.getSimpleName());
			}
			Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
			// 通过反射获取自己及父接口中的所有方法然后遍历之
			for (Method method : targetType.getMethods()) {
				// 以下几种情况跳过不处理
				if (method.getDeclaringClass() == Object.class || // Object类中的方法、
					(method.getModifiers() & Modifier.STATIC) != 0 || // static修饰的方法
					Util.isDefault(method)) { // jdk8的default方法
					// 跳过不处理
					continue;
				}
				
				// 关键代码：解析该方法的契约（注解），如果该方法没有@RequestLine注解，解析时会抛出异常
				MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
				
				// 一个类中的方法不能有相同的configKey
				checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s", metadata.configKey());
				// 存储configKey与方法元数据的对应关系
				result.put(metadata.configKey(), metadata);
			}
		
			// 返回方法的所有元数据列表（不包括static、default、Object类中的方法）
			return new ArrayList<>(result.values());
		}
		
		/**
		* 解析feign api接口上中的某个方法上的所有注解及类上注解（类上的注解作用于所有方法）
		*/
		protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
			MethodMetadata data = new MethodMetadata();
			// 设置方法的返回值类型
			data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
			//设置方法的签名（例如UserFeignClient#list(User,Map)）
			data.configKey(Feign.configKey(targetType, method));
			// 只有1个父接口
			if (targetType.getInterfaces().length == 1) {
				// 处理父接口上的注解（@Headers）
				processAnnotationOnClass(data, targetType.getInterfaces()[0]);
			}
			 // 处理Class上的注解(@Headers)
			processAnnotationOnClass(data, targetType);
			
			
			// 一个方法上可能有多个注解（@RequestLine、@Body、@Headers）
			for (Annotation methodAnnotation : method.getAnnotations()) {
				//处理方法上的注解（@RequestLine、@Body、@Headers）
				processAnnotationOnMethod(data, methodAnnotation, method);
			}
			// 方法上没有@RequestLine注解，或者注解没有GET、POST这种method时抛出异常
			checkState(data.template().method() != null, 
					   "Method %s not annotated with HTTP method type (ex. GET, POST)", method.getName());
			
			// 获得所有参数的Class对象
			Class<?>[] parameterTypes = method.getParameterTypes();
			// 获得参数化的参数泛型类型
			Type[] genericParameterTypes = method.getGenericParameterTypes();
 			// 获得方法的所有参数的注解，一个参数可能有多个注解，所以是个二维数组
			// 如果某个参数没有注解，则该参数的注解是个空数组
			Annotation[][] parameterAnnotations = method.getParameterAnnotations();
			// 方法的参数个数
			int count = parameterAnnotations.length;
			for (int i = 0; i < count; i++) {
				// 参数是否有@Param、@QueryMap、@HeaderMap这几个http注解中的任意1个
				boolean isHttpAnnotation = false;
				if (parameterAnnotations[i] != null) { //不清楚为什么要判断不为空，难道可能为空？？
					// 处理某个参数上的多个注解（1个参数可能有多个注解）
					isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
				}
				
				// 如果当前遍历的参数的类型是URI.class
				if (parameterTypes[i] == URI.class) {
					// 记录URI类型参数在参数列表中的索引，参数类型是URI，后面构造http请求时使用该URI，
					// 而不是使用构建Feign.builder().target(xx, 'http://xxx')时的url
					data.urlIndex(i);
				} 
				// 当前遍历的参数没有@Param、@QueryMap、@HeaderMap这几个http注解，
				// 且参数类型不是Request.Options.class， 那么该参数就是http请求的body部分
				else if (!isHttpAnnotation && parameterTypes[i] != Request.Options.class) {
					// 这里存在参数顺序的问题：
					// 1. 先有表单提交参数(表单参数是有条件的，指没有被表达式使用的@Param参数)，就不能再有body参数，
					//    eg: void postUser1(@Param("user_name") String username, User user);是错误的
					// 2.先有body参数，后面可以有表单提交参数(没有被表达式使用的@Param参数)
					//    eg: void postUser2(User user, @Param("user_name") String username);是正确的
					checkState(data.formParams().isEmpty(),
							   "Body parameters cannot be used with form parameters.");
					// 检查是否已经设置过http body参数，即不能有多个htp body参数, 否则抛出异常
					// 例如 void postUser3(User user, Address address);是错误的，因为有多个参数作为body请求体
					checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
					// 记录作为body请求体参数的位置
					data.bodyIndex(i);
					// 这行代码用于记录body参数的实际类型
					// Types.resolve(targetType, targetType, genericParameterTypes[i])用于获取该参数的实际类型
					// 例如泛型参数则返回实际参数类型，示例如下：
					/**
					*  interface BaseApi<V> {
					*      void put(@Param("key") String key, V value);
					* }
					*
					* public interface Api extends BaseApi<User> {
					*
					* }
					*
					* 当调用api.put("caleb", user);方法时，此时 BaseApi中的put方法的第2个参数value作为body，
					* Types.resolve(Api.class, Api.class,  vlaue)得到的value的类型是User.class类型
					*/
					data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
				}
			}

			if (data.headerMapIndex() != null) {
				checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()],
							   genericParameterTypes[data.headerMapIndex()]);
			}

			if (data.queryMapIndex() != null) {
				if (Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()])) {
					checkMapKeys("QueryMap", genericParameterTypes[data.queryMapIndex()]);
				}
			}

			return data;
		}
		
		/**
		* 处理类上的注解， 默认由子类Contract.Default实现
		/
		protected abstract void processAnnotationOnClass(MethodMetadata data, Class<?> clz);

		/**
		* 处理方法上的注解， 默认由子类Contract.Default实现
		*/
		protected abstract void processAnnotationOnMethod(MethodMetadata data,
											Annotation annotation,
											Method method);
		/**
		* 处理参数上的注解， 默认由子类Contract.Default实现
		*/
		protected abstract boolean processAnnotationsOnParameter(MethodMetadata data,
											Annotation[] annotations,
											int paramIndex);
	
		protected void nameParam(MethodMetadata data, String name, int i) {
			Collection<String> names =
				data.indexToName().containsKey(i) ? data.indexToName().get(i) : new ArrayList<String>();
			names.add(name);
			data.indexToName().put(i, names);
		}
	}
	
	/**
	* 契约的默认实现，spring-cloud-opengeign的实现为org.springframework.cloud.openfeign.support.SpringMvcContract
	*/
	class Default extends BaseContract {
		/**
		* 处理类上的注解（@Headers）
		*/
		@Override
		protected void processAnnotationOnClass(MethodMetadata data, Class<?> targetType) {
			//判断类上是否存在@Headers注解
			if (targetType.isAnnotationPresent(Headers.class)) {
				//获得@Headers注解的value属性的值
				String[] headersOnType = targetType.getAnnotation(Headers.class).value();
				//检查注解是否有值，无值则抛出异常
				checkState(headersOnType.length > 0, "Headers annotation was empty on type %s.",
						   targetType.getName());
				// 把注解的值(http请求头)用冒号(:)分割，因为http允许重复的header
				// (比如set-cookie:uid=12、set-cookie:sid=22)，所以一个name对应多个value， 
				// 这里的headers这个Map对象的key就是http header的name， 值就是http header的
				// 多个相同header的值集合
				Map<String, Collection<String>> headers = toMap(headersOnType);
				//将metadata中原有headers放到新headers中
				headers.putAll(data.template().headers());
				data.template().headers(null); 
				//将metadata的headers属性替换为新的headers
				data.template().headers(headers);
			}
		}

		/**
		* 处理方法上的注解（@RequestLine、@Body、@Headers）
		*/
		@Override
		protected void processAnnotationOnMethod(MethodMetadata data, Annotation methodAnnotation, Method method) {
			Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
			// 判断是否是@RequestLine注解
			if (annotationType == RequestLine.class) {
				//获取@RequestLine注解的value属性的值
				String requestLine = RequestLine.class.cast(methodAnnotation).value();
				// @RequestLine注解的value属性的值若为空，则抛出异常
				checkState(emptyToNull(requestLine) != null,
						   "RequestLine annotation was empty on method %s.", method.getName());
				// 正则表达式匹配注解的值
				Matcher requestLineMatcher = REQUEST_LINE_PATTERN.matcher(requestLine);
				//判断@RequestLine注解的value属性的值是否匹配正则表达式
				if (!requestLineMatcher.find()) {
					// @RequestLine注解的value属性的值不匹配给定的正则表达式，抛出异常
					throw new IllegalStateException(String.format(
						"RequestLine annotation didn't start with an HTTP verb on method %s",
						method.getName()));
				} 
				else {
					// 获取@RequestLine注解value属性值的的method部分，例如GET /user/list获取到的就是GET
					data.template().method(HttpMethod.valueOf(requestLineMatcher.group(1)));
					// 获取url部分，对于上一行的示例获取到的就是/user/list
					data.template().uri(requestLineMatcher.group(2));
				}
				 // 是否将'%2F'反转为'/'
				data.template().decodeSlash(RequestLine.class.cast(methodAnnotation).decodeSlash());
				// 参数集合格式化方式，默认使用key=value0&key=value1
				data.template().collectionFormat(RequestLine.class.cast(methodAnnotation).collectionFormat());

			}
			// 判断处理的注解是否是@Body
			else if (annotationType == Body.class) {
				String body = Body.class.cast(methodAnnotation).value();
				// @Body注解的value必须有内容(不能为空字符串)
				checkState(emptyToNull(body) != null, "Body annotation was empty on method %s.",
						   method.getName());
				// 是否有{这种表达式(模板变量)
				if (body.indexOf('{') == -1) {
					// @Body注解的内容不包含需要解析的变量，不需要额外处理
					data.template().body(body);
				} else {
					// 是模板，含有表达式变量，{expression}这种表达式需要插值替换成成真正的值
					//（用方法的参数值替换）
					data.template().bodyTemplate(body);
				}
			} 
			//判断处理的注解是否是@Headers注解
			else if (annotationType == Headers.class) {
				String[] headersOnMethod = Headers.class.cast(methodAnnotation).value();
				//@Headers注解的value属性必须有值，若无值则抛出异常
				checkState(headersOnMethod.length > 0, "Headers annotation was empty on method %s.",
						   method.getName());
				// 由于类和方法上都可以有@Headers注解，所以内部实现（appendHeader）是合并
				// 之前已有的headers和这里新增的headers
				data.template().headers(toMap(headersOnMethod));
			}
		}

		/**
		* 处理某个参数上的多个注解（1个参数可能有多个注解）,
		* 总结，主要处理了如下几个属性:
		* indexToName            --> @Param注解的value属性
		* indexToExpanderClass   --> @Param注解的expander属性
		* indexToEncoded         --> @Param注解的encoded属性
		* formParams             --> @Param注解的value属性
		* queryMapIndex          --> @QueryMap注解
		* queryMapEncoded        --> @QueryMap注解的encoded属性
		* headerMapIndex         --> @HeaderMap注解
		
		*
		* @param data 元数据，存储方法的相关数据(uri、headers、queries、body。。。)
		* @param annotations 某1个参数上的所有所有注解（1个参数可能有多个注解）
		* @param paramIndex 该参数在方法的所有参数中的索引, 从0开始
		*/
		@Override
		protected boolean processAnnotationsOnParameter(MethodMetadata data, 
									Annotation[] annotations,
									int paramIndex) {
			// 参数是否有@Param、@QueryMap、@HeaderMap这几个http注解
			boolean isHttpAnnotation = false;
			
			for (Annotation annotation : annotations) {
				// 获取注解的Class对象
				Class<? extends Annotation> annotationType = annotation.annotationType();
				// 如果处理的是@Param注解
				if (annotationType == Param.class) {
					Param paramAnnotation = (Param) annotation;
					// 获得@Param注解的value属性值
					String name = paramAnnotation.value();
					// value属性值不能为空
					checkState(emptyToNull(name) != null, "Param annotation was empty on param %s.", 
							   paramIndex);
					// 追加（不是替换indexToName）到MethodMetadata的indexToName属性中，
					// key为参数位置索引， values为@Param注解的value属性值
					nameParam(data, name, paramIndex);
					// @Param注解的expander参数，定义参数的解释器，默认是ToStringExpander，
					// 调用参数的toString方法
					Class<? extends Param.Expander> expander = paramAnnotation.expander();
					// 判断是否自定义了expander解析器
					if (expander != Param.ToStringExpander.class) {
						// 参数自定义了expander解析器，存储起来
						data.indexToExpanderClass().put(paramIndex, expander);
					}
					// 参数是否已经urlEncoded，如果没有，会使用urlEncoded方式编码
					data.indexToEncoded().put(paramIndex, paramAnnotation.encoded());
					// 参数上存在http注解
					isHttpAnnotation = true;
					if (!data.template().hasRequestVariable(name)) {
						// 如果参数不在path里面，不在query里面，不在header里面，就设置到formParam中
						data.formParams().add(name);
					}
				} 
				// 如果处理的是@QueryMap注解，注解参数对象时，将该参数转换为http请求参数格式发送
				else if (annotationType == QueryMap.class) {
					// 1个方法不能在多个参数上存在@QueryMap注解
					checkState(data.queryMapIndex() == null,
							   "QueryMap annotation was present on multiple parameters.");
					// 记录@QueryMap注解参数的索引
					data.queryMapIndex(paramIndex);
					// @QueryMap注解的对像的值是否已经编码过，如果未编码则后续将url编码
					data.queryMapEncoded(QueryMap.class.cast(annotation).encoded());
					// 参数上存在http注解
					isHttpAnnotation = true;
				}
				// 如果处理的是@HeaderMap注解，注解一个Map类型的参数，放入http header中发送
				else if (annotationType == HeaderMap.class) {
					// 1个方法中不能在多个参数上存在@HeaderMap注解
					checkState(data.headerMapIndex() == null,
							   "HeaderMap annotation was present on multiple parameters.");
					// 记录@HeaderMap注解参数的索引
					data.headerMapIndex(paramIndex);
					// 参数上存在http注解
					isHttpAnnotation = true;
				}
			}
			
			// 参数是否有@Param、@QueryMap、@HeaderMap这几个http注解
			return isHttpAnnotation;
		}

		/**
		* 将http header用冒号分割
		* 例如 Content-Type:text/html、Set-Cookie:uid=123、Set-Cookie:sid=456将变成如下的Map对象
		* Content-Type   ---->   text/html
		* Set-Cookie     ---->   uid=123、sid=456
		*/
		private static Map<String, Collection<String>> toMap(String[] input) {
			Map<String, Collection<String>> result = new LinkedHashMap<String, Collection<String>>(input.length);
			for (String header : input) {
				int colon = header.indexOf(':');
				String name = header.substring(0, colon);
				if (!result.containsKey(name)) {
					result.put(name, new ArrayList<String>(1));
				}
				result.get(name).add(header.substring(colon + 1).trim());
			}
			return result;
		}
	}
}
```

代码稍微有点多，但是逻辑很清晰，先处理类上的注解，再处理方法上注解，最后处理方法参数注解，把所有注解的情况都处理到就可以了。

生成的MethodMetadata的结构如下：
```java
public final class MethodMetadata implements Serializable {
	
	//标识方法的key，接口名加方法签名：GitHub#contributors(String,String)
	private String configKey;
	//方法返回值类型
	private transient Type returnType;
	//uri参数的位置，方法中可以提供1个java.net.URI类型的参数，发请求时直接使用这个参数
	private Integer urlIndex;
	//body参数的位置，只能有一个未注解的参数为body，否则报错
	private Integer bodyIndex;
	//@HeaderMap注解参数的位置
	private Integer headerMapIndex;
	//@QueryMap注解参数位置
	private Integer queryMapIndex;
	//@QueryMap注解里面encode参数，是否已经urlEncode编码过了
	private boolean queryMapEncoded;
	//body参数的类型
	private transient Type bodyType;
	//RequestTemplate 原型
	private RequestTemplate template = new RequestTemplate();
	//form请求参数
	private List<String> formParams = new ArrayList<String>();
	//方法参数位置和名称的map
	private Map<Integer, Collection<String>> indexToName ;
	//@Param中注解的expander方法，可以指定解析参数类
	private Map<Integer, Class<? extends Expander>> indexToExpanderClass ;
	//参数是否被urlEncode编码过了，@Param中encoded方法
	private Map<Integer, Boolean> indexToEncoded ;
	//自定义的Expander
	private transient Map<Integer, Expander> indexToExpander;
	
	...省略部分代码
}
```

>```Contract```也是feign的一个扩展点，一个优秀组件的架构通常是具有很强的扩展性，feign的架构本身很简单，设计的扩展点也很简单方便，所以受到spring的青睐，将其集成到spring cloud中。spring cloud就是通过```Contract```的扩展(```org.springframework.cloud.openfeign.support.SpringMvcContract```)，实现使用springMVC的注解接入feign。feign自己还实现了使用jaxrs注解接入feign。

# 初始化总结
上文已经完成了feign初始化结构为动态代理的整个过程，简单的捋一遍：

1. 初始化```Feign.Builder```传入参数，构造```ReflectiveFeign```
2. ```ReflectiveFeign```通过内部类```ParseHandlersByName```的```Contract```属性，解析接口生成```MethodMetadata```
3. ```ParseHandlersByName```根据```MethodMetadata```生成```RequestTemplate```工厂
4. ```ParseHandlersByName```创建```SynchronousMethodHandler```，传入```MethodMetadata```、```RequestTemplate```工厂和```Feign.Builder```相关参数
5. ```ReflectiveFeign```创建```FeignInvocationHandler```，传入参数```SynchronousMethodHandler```，绑定```DefaultMethodHandler```
6. ```ReflectiveFeign```根据```FeignInvocationHandler```创建```Proxy```

关键的几个类是：

- ```ReflectiveFeign``` 初始化入口
- ```FeignInvocationHandler``` 实现动态代理的InvocHandler
- ```SynchronousMethodHandler``` 方法处理器，方法调用处理器
- ```MethodMetadata``` 方法元数据
- ```Contract.Default```契约解析默认实现

# 接口调用
为方便理解，分析完feign源码后，我将feign执行过程分成三层，如下图：
![](https://oscimg.oschina.net/oscnet/up-9d5d9d57db923dc0643764f2db79c6a9e5a.png)

三层分别为：

- 代理层  jdk动态代理调用层
- 转换层 方法转http请求，解码http响应
- 网络层 http请求发送

java动态代理接口方法调用，会调用到InvocaHandler的invoke方法，feign里面实现类是FeignInvocationHandler，invoke代码如下：
```java
public class ReflectiveFeign extends Feign {
	
	...省略部分代码
		
	static class FeignInvocationHandler implements InvocationHandler {
		// 要代理的feign api接口，包含class、host信息
		private final Target target;
		// MethodHandler的具体类型位SynchronousMethodHandler或DefaultMethodHandler
		private final Map<Method, MethodHandler> dispatch;

		FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
			this.target = checkNotNull(target, "target");
			this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
		}

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			...省略 部分代码

			return dispatch.get(method).invoke(args);
		}
	}
}
```

根据方法找到```MethodHandler```，除接口的```default```方法外(```default```方法直接调用，无需feign处理)，找到的是```SynchronousMethodHandler```对象，然后调用```SynchronousMethodHandlerd.invoke```方法：
```java
final class SynchronousMethodHandler implements MethodHandler {
	private final MethodMetadata metadata;
	private final Target<?> target;
	private final Client client;
	private final Retryer retryer;
	private final List<RequestInterceptor> requestInterceptors;
	private final Logger logger;
	private final Logger.Level logLevel;
	private final RequestTemplate.Factory buildTemplateFromArgs;
	private final Options options;
	private final Decoder decoder;
	private final ErrorDecoder errorDecoder;
	private final boolean decode404;
	private final boolean closeAfterDecode;
	private final ExceptionPropagationPolicy propagationPolicy;

	@Override
	public Object invoke(Object[] argv) throws Throwable {
		//buildTemplateFromArgs是RequestTemplate工程对象，根据方法参数创建RequestTemplate
		RequestTemplate template = buildTemplateFromArgs.create(argv);
		Options options = findOptions(argv);
		//重试设置
		Retryer retryer = this.retryer.clone();
		while (true) {
			try {
				//执行和解码
				return executeAndDecode(template, options);
			} 
			catch (RetryableException e) {
				try {
					// 执行executeAndDecode时发生异常，由retryer决定是否继续重试
					//（若continueOrPropagate的实现抛出异常则停止重试，否则继续重试）
					retryer.continueOrPropagate(e);
				}
				catch (RetryableException th) {
					// continueOrPropagate的实现抛出了异常，停止重试
					Throwable cause = th.getCause();
					if (propagationPolicy == UNWRAP && cause != null) {
						throw cause;
					} 
					else {
						throw th;
					}
				}
				
				if (logLevel != Logger.Level.NONE) {
					logger.logRetry(metadata.configKey(), logLevel);
				}
				continue;
			}
		}
	}

	Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
		//RequestTemplate转换为Request
		Request request = targetRequest(template);

		...省略部分代码

		Response response;
		long start = System.nanoTime();
		try {
			response = client.execute(request, options);
		} 
		catch (IOException e) {
			...省略部分代码
			throw errorExecuting(request, e);
		}
		long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

		boolean shouldClose = true;
		try {
			...省略部分代码
				
			  //如果接口方法返回的是Response类	
			if (Response.class == metadata.returnType()) {
				if (response.body() == null) {
					return response;
				}
				//body不为空，且length>最大缓存值，返回response，但是不能关闭response
				if (response.body().length() == null ||
					response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
					shouldClose = false;
					return response;
				}
				// body的length小于最大缓存值，可以直接读取body字节数组，返回response，可以自动关闭response
				byte[] bodyData = Util.toByteArray(response.body().asInputStream());
				return response.toBuilder().body(bodyData).build();
			}
			//判断响应成功
			if (response.status() >= 200 && response.status() < 300) {
				// 响应成功，方法是否没有返回值
				if (void.class == metadata.returnType()) {
					// 方法返回void类型
					return null;
				} 
				// 方法需要返回值
				else {
					// 解码响应，直接调用decoder解码
					Object result = decode(response);
					//是否应该自动关闭response响应
					shouldClose = closeAfterDecode;
					// 方法最终的返回值
					return result;
				}
			} 
			// 请求返回了404状态码并且应该解码404响应，并且方法需要返回值
			else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
				// 解码404状态的response响应，直接调用decoder解码
				Object result = decode(response);
				shouldClose = closeAfterDecode;
				return result;
			} 
			else {
				// 非2xx的状态或者404状态不解码时，使用errorDecoder解析要抛出的异常
				throw errorDecoder.decode(metadata.configKey(), response);
			}
		}
		catch (IOException e) {
			...省略部分代码
			throw errorReading(request, response, e);
		}
		finally {
			 //是否需要关闭response，根据Feign.Builder 参数设置是否要关闭流
			if (shouldClose) {
				//关闭响应
				ensureClosed(response.body());
			}
		}
	}

	Request targetRequest(RequestTemplate template) {
		for (RequestInterceptor interceptor : requestInterceptors) {
			interceptor.apply(template);
		}
		return target.apply(template);
	}
}
```
过程比较简单，生成RquestTemplate -> 转换为Request -> client发请求 -> Decoder解析Response

# RquestTemplate构建过程
先看看RequestTemplate的结构：
```java
public final class RequestTemplate implements Serializable {
	
	private static final Pattern QUERY_STRING_PATTERN = Pattern.compile("(?<!\\{)\\?");
	// 请求参数 ?后面的name=value
	private final Map<String, QueryTemplate> queries = new LinkedHashMap<>();
	// 请求头
	private final Map<String, HeaderTemplate> headers = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
	// 基础url部分
	private String target;
	// url中的#及其后面的部分
	private String fragment;
	private boolean resolved = false;
	// @RequestLine注解中的请求路径对象，只有路径部分(/user/list)，不包含?后的部分
	private UriTemplate uriTemplate;
	//请求方法 GET/POST等
	private HttpMethod method;
	//字符集
	private transient Charset charset = Util.UTF_8;
	//请求体
	private Request.Body body = Request.Body.empty();
	//是否decode斜杠，将"%2F"反转为"/"
	private boolean decodeSlash = true;
	//集合格式化，a=b&c=d
	private CollectionFormat collectionFormat = CollectionFormat.EXPLODED;

}
```

在```SynchronousMethodHandler.invoke```方法中生成```RequestTemplate```
```java
//buildTemplateFromArgs是RequestTemplate.Factory实现类
RequestTemplate template = buildTemplateFromArgs.create(argv);
```

```RequestTemplate.Factory```有三个实现类：
- ```BuildTemplateByResolvingArgs``` ```RequestTemplate```工厂
- ```BuildEncodedTemplateFromArgs``` ```BuildTemplateByResolvingArgs```的子类 重载```resolve```方法，解析form表单请求
- ```BuildFormEncodedTemplateFromArgs``` ```BuildTemplateByResolvingArgs```的子类，重载```resolve```方法，解析body请求
----

- **BuildTemplateByResolvingArgs的实现**

默认的RequestTemplate的工厂，没有请求体，不需要编码器

```java
private static class BuildTemplateByResolvingArgs implements RequestTemplate.Factory {
	// @QueryMap注解的参数的编码器
	private final QueryMapEncoder queryMapEncoder;
	// 方法的元数据
	protected final MethodMetadata metadata;
	// @Param注解的expander属性，用于参数转换
	private final Map<Integer, Expander> indexToExpander = new LinkedHashMap<Integer, Expander>();

	/**
	* @params argv 调用方法时传递的参数列表
	*/
	@Override
	public RequestTemplate create(Object[] argv) {
		RequestTemplate mutable = RequestTemplate.from(metadata.template());
		// 方法中有java.net.URI类型的参数, 例如void list(String username, URI url, String queryKey)
		if (metadata.urlIndex() != null) {
			// 获取URI类型的参数在参数列表中的位置
			int urlIndex = metadata.urlIndex();
			// URI类型的参数的实际值不为null
			checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
			// 使用指定的参数的地址作为请求的基础地址
			//（不使用Feign.builder().target(Api.class, 'http://localhost:8080')）中的url地址
			mutable.target(String.valueOf(argv[urlIndex]));
		}
		
		// 存储  @Param注解value值 --> 参数实际值  的映射关系，用于后续表达式{expression}的求值
		Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
		// indexToName属性是解析@Param注解产生的， 
		// key为@Param注解的参数在参数列表中的索引
		// value为@Param注解的value属性
		for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
			// 获得@Param主键的参数索引
			int i = entry.getKey();
			// 该位置参数的实际调用时传递的值
			Object value = argv[entry.getKey()];
			if (value != null) { // 跳过null值
				// @Param注解是否有expander属性，即参数值是否需要转换处理（如Date -> String、Enum -> int等等）
				if (indexToExpander.containsKey(i)) {
					// 使用expander属性指定的class来转换输入值为另一个值
					//（例如传入Date类型的参数，实际请求时需要的是yyyy-MM-dd这种格式的字符串，
					// 那么就需要写一个DateFormatExpander类来进行转换参数）
					value = expandElements(indexToExpander.get(i), value);
				}
				for (String name : entry.getValue()) {
					// 存储@Param注解value值 --> 参数实际值  的映射关系，用于后续表达式{expression}的求值
					varBuilder.put(name, value);
				}
			}
		}

		//解析RequestTemplate
		RequestTemplate template = resolve(argv, mutable, varBuilder);
		// 为什么单独把queryMap放在这里解析，而不是在resolve方法中，或者在RequestTemplate中？(该问题来源于拍拍贷的博客内容)
		// 因为@QueryMap注解的参数不需要解析表达式
		if (metadata.queryMapIndex() != null) { //判断是否有@QueryMap注解的参数
			// 获取@QueryMap注解的参数的实际值
			Object value = argv[metadata.queryMapIndex()];
			// value可能是Map或者java bean对象，如果参数不是Map类型的则用queryMapEncoder进行编码
			Map<String, Object> queryMap = toQueryMap(value);
			// 合并已有的查询参数和queryMap的参数，追加到RequestTemplate的queries属性中
			template = addQueryMapQueryParameters(queryMap, template);
		}
		
		// 为什么单独把headerMap放在这里解析，而不是在resolve方法中，或者在RequestTemplate中？
		// 因为@HeaderMap注解的参数不需要解析表达式
		// 参数中是否有@HeaderMap注解的参数
		if (metadata.headerMapIndex() != null) {
			// 将@HeaderMap注解的参数的实际值与已有的(@Headers的)header参数合并，
			// 追加到RequestTemplate的headers属性中
			template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
		}

		return template;
	}
	
	/**
	* 注解的表达书求值
	*
	* @params variables 指的是@Param注解value值 --> 参数实际值  的映射关系，用于后续表达式{expression}的求值
	*/
	protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable, Map<String, Object> variables) {
		// 由RequestTemplate.resolve(Map<String, Object>)实现，真正的去解析注解中的表达式{expression}
		return mutable.resolve(variables);
	}
}
```
- **BuildFormEncodedTemplateFromArgs的实现**

 如果有formParam，并且bodyTemplate不为空，请求体为x-www-form-urlencoded格式

 将会解析form参数，填充到bodyTemplate中

该类用于对参数列表中的form参数(**未被表达式使用**的```@Param```参数)使用encoder编码

```java
private static class BuildFormEncodedTemplateFromArgs extends BuildTemplateByResolvingArgs {
	// Feign.builder().encoder(xx)时设置的编码器
	private final Encoder encoder;
	
	...省略部分代码

	/**
	* 注意这里重写了父类BuildTemplateByResolvingArgs中的resolve方法， 
	* create的逻辑仍然使用父类的，这里是使用了模板方法设计模式
	* @param variables  指参数上的@Param注解的键值对
	*/
	@Override
	protected RequestTemplate resolve(Object[] argv,
							RequestTemplate mutable,
							Map<String, Object> variables) {
		
		// 存储所有的form表单参数的name、value
		Map<String, Object> formVariables = new LinkedHashMap<String, Object>();
		
		for (Entry<String, Object> entry : variables.entrySet()) {
			// 判断该参数是否是form表单参数，为什么有这个判断?
			//  因为@Param注解的参数作为form表单参数是有条件的，不是所有的@Param注解的参数都会作为form表单参数
			// @Param注解的参数有2种作用：
			// 1. 如果@Param注解的参数被表达式所使用，则不会作为form参数, 仅仅是为了解析表达式变量使用的
			// 2. 如果@Param注解的参数没有被表达式所使用，则作为form表单参数
			if (metadata.formParams().contains(entry.getKey())) {
				// 拿到所有的form表单参数的name、value
				formVariables.put(entry.getKey(), entry.getValue());
			}
		}
		try {
			// 使用encoder对参数编码
			encoder.encode(formVariables, Encoder.MAP_STRING_WILDCARD, mutable);
		}
		catch (EncodeException e) {
			throw e;
		} 
		catch (RuntimeException e) {
			throw new EncodeException(e.getMessage(), e);
		}
		//调用父类BuildTemplateByResolvingArgs.reslove()处理，对注解中的表达式求值
		return super.resolve(argv, mutable, variables);
	}
}
```

- **BuildEncodedTemplateFromArgs的实现**

如果包含请求体，将会用encoder编码请求体对象（参数列表中的参数**无任何注解**会作为请求体）

```java
private static class BuildEncodedTemplateFromArgs extends BuildTemplateByResolvingArgs {
	// Feign.builder().encoder(xx)时设置的编码器
	private final Encoder encoder;
	
	...省略部分代码

	@Override
	protected RequestTemplate resolve(Object[] argv,
						RequestTemplate mutable,
						Map<String, Object> variables) {
		// 获取作为body的参数的值
		Object body = argv[metadata.bodyIndex()];
		// 值不能为空
		checkArgument(body != null, "Body parameter %s was null", metadata.bodyIndex());
		try {
			// 编码并设置RequestTemplate的body
			encoder.encode(body, metadata.bodyType(), mutable);
		}
		catch (EncodeException e) {
			throw e;
		} 
		catch (RuntimeException e) {
			throw new EncodeException(e.getMessage(), e);
		}
		//调用父类BuildTemplateByResolvingArgs.reslove()处理，对注解中的表达式求值
		return super.resolve(argv, mutable, variables);
	}
}
```

# RequestTemplate解析参数的方法：
```java
public final class RequestTemplate implements Serializable {
		private static final Pattern QUERY_STRING_PATTERN = Pattern.compile("(?<!\\{)\\?");
	// 请求参数 ?后面的name=value
	private final Map<String, QueryTemplate> queries = new LinkedHashMap<>();
	// 请求头
	private final Map<String, HeaderTemplate> headers = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
	// 基础url部分
	private String target;
	// url中的#及其后面的部分
	private String fragment;
	private boolean resolved = false;
	// @RequestLine注解中的请求路径对象，只有路径部分(/user/list)，不包含?后的部分
	private UriTemplate uriTemplate;
	//请求方法 GET/POST等
	private HttpMethod method;
	//字符集
	private transient Charset charset = Util.UTF_8;
	//请求体
	private Request.Body body = Request.Body.empty();
	//是否decode斜杠，将"%2F"反转为"/"
	private boolean decodeSlash = true;
	//集合格式化，a=b&c=d
	private CollectionFormat collectionFormat = CollectionFormat.EXPLODED;
	
	/**
	* 表达式{expression}解析
	*
	* @param variables 所有的@Param注解value值 --> 参数实际值  的映射关系，用于表达式{expression}求值
	*/
	public RequestTemplate resolve(Map<String, ?> variables) {

		StringBuilder uri = new StringBuilder();
		// 得到当前对象一份新的拷贝
		RequestTemplate resolved = RequestTemplate.from(this);
		// 为什么要判断是否非null？, 所有的方法必须要有@RequestLine注解啊，
		// 即使如@RequestLine("GET")这样不写path这里的uriTemplate也会是空字符串""，而不可能是null啊
		if (this.uriTemplate == null) {
			this.uriTemplate = UriTemplate.create("", !this.decodeSlash, this.charset);
		}
		// 解析@RequestLine的属性中的表达式为实际值, 此时的uri为具体url，
		// 例如 /user/{username} 解析后变成 /user/calebzhao
		uri.append(this.uriTemplate.expand(variables));

		// 判断@RequestLine("GET /user/{username}")是否有查询参数, 注意此时@QueryMap的参数还不在queries属性中
		// 为什么不把@QueryMap的处理放在这个方法里就是因为@QueryMap的参数不需要解析表达式变量
		if (!this.queries.isEmpty()) {
			// 初始化1个空Map
			resolved.queries(Collections.emptyMap());
			StringBuilder query = new StringBuilder();
			Iterator<QueryTemplate> queryTemplates = this.queries.values().iterator();

			while (queryTemplates.hasNext()) {
				QueryTemplate queryTemplate = queryTemplates.next();
				// 解析@QueryMap注解标注的参数的值，然后变成查询字符串，格式如a=b
				String queryExpanded = queryTemplate.expand(variables);
				// 有值，需要拼接在url中
				if (Util.isNotBlank(queryExpanded)) {
					query.append(queryExpanded);
					if (queryTemplates.hasNext()) {
						// 多个url参数之间的分隔符，如a=b&c=d
						query.append("&");
					}
				}
			}

			String queryString = query.toString();
			if (!queryString.isEmpty()) {
				Matcher queryMatcher = QUERY_STRING_PATTERN.matcher(uri);
				// 判断uri中是否已有查询字符串?
				if (queryMatcher.find()) {
					// @RequestLine("GET /user/{username}?appId=123")中的url路径已有查询字符串?
					uri.append("&");
				} 
				.// 还没有字符串?，则拼接?
				else {
					uri.append("?");
				}
				// 拼接查询参数到url中
				uri.append(queryString);
			}
		}
		
		// 到这里uri已经变成/user/calebzhao?appId=123&a=b&c=d这种形式了

		// 将上述uri的各部分添加到当前RequestTemplate中，
		// 重新计算当前RequestTemplate的各个部分的值(uri路径覆盖当前值、queries查询参数覆盖当前值、frament覆盖)
		// RequestTemplate的各个部分的值如下：
		// uri  ---> /user/calebzhao
		// quries ---> appId=123、appId=123、a=b、c=d
		// frament --->空
		resolved.uri(uri.toString());

		// 有@Headers注解的值，注意此时@HeaderMap注解的参数值还不在headers，
		// 为什么不把@HeaderMap的处理放在这个方法里就是因为因为@HeaderMap注解的参数值不需要解析表达式
		if (!this.headers.isEmpty()) {
			// header属性初始化为空Map
			resolved.headers(Collections.emptyMap());
			for (HeaderTemplate headerTemplate : this.headers.values()) {
				// HeaderTemplate的template形如：name空格value1逗号空格value2 ，
				// 例如set-cookie rid={roleId}, sid={sessionId}
				
				// 解析@Headers("x-user:{username}")注解中的表达式{username}
				String header = headerTemplate.expand(variables);
				if (!header.isEmpty()) {
					// 得到的值形如：set-cookie rid=1, sid=2
					// 得到第一个空格后面的部分，例如上面例子的rid=1, sid=2
					String headerValues = header.substring(header.indexOf(" ") + 1);
					if (!headerValues.isEmpty()) {
						// 存储解析后的header
						resolved.header(headerTemplate.getName(), headerValues);
					}
				}
			}
		}

		// 处理body中的表达式
		resolved.body(this.body.expand(variables));

		//标记为已解析
		resolved.resolved = true;
		//返回解析后的RequestTemplate
		return resolved;
	}
	
	/**
	* 这个方法的作用就是复制requestTemplate，得到一份新的拷贝
	*/
	public static RequestTemplate from(RequestTemplate requestTemplate) {
		RequestTemplate template = new RequestTemplate(requestTemplate.target, requestTemplate.fragment,
								requestTemplate.uriTemplate,
								requestTemplate.method, requestTemplate.charset,
								requestTemplate.body, requestTemplate.decodeSlash, 
								requestTemplate.collectionFormat);

		if (!requestTemplate.queries().isEmpty()) {
			template.queries.putAll(requestTemplate.queries);
		}

		if (!requestTemplate.headers().isEmpty()) {
			template.headers.putAll(requestTemplate.headers);
		}
		return template;
	}
}
```

我们回到SynchronousMethodHandler的invoke方法继续看executeAndDecode的实现
```java
 @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
	  
	...省略
	return executeAndDecode(template, options);
 }
```
# RquestTemplate转换Request

先来看看Request的结构，完整的http请求信息的定义：
```java
public final class Request {
	
	private final HttpMethod httpMethod;
	private final String url;
	private final Map<String, Collection<String>> headers;
	private final Body body;

	public static class Body {

		private final byte[] data;
		private final Charset encoding;
		private final BodyTemplate bodyTemplate;
	}
	
	Request(HttpMethod method, String url, Map<String, Collection<String>> headers, Body body) {
		this.httpMethod = checkNotNull(method, "httpMethod of %s", method.name());
		this.url = checkNotNull(url, "url");
		this.headers = checkNotNull(headers, "headers of %s %s", method, url);
		this.body = body;
	}
}
```
SynchronousMethodHandler的targetRequest方法将RequestTemplate转换为Request
```java
Request targetRequest(RequestTemplate template) {
	//先应用所用拦截器，拦截器是在Feign.Builder中传入的，拦截器可以修改RequestTemplate信息
	for (RequestInterceptor interceptor : requestInterceptors) {
		interceptor.apply(template);
	}
	//调用Target的apply方法，默认Target是HardCodedTarget
	return target.apply(template);
}
```
这块先应用所有拦截器，然后target的apply方法。拦截器和target都是扩展点，拦截器可以在构造好RequestTemplate后和发请求前修改请求信息，target默认使用HardCodedTarget直接发请求，feign还提供了LoadBalancingTarget，适配Ribbon来发请求，实现客户端的负载均衡。
- ```HardCodedTarget```
```java
public static class HardCodedTarget<T> implements Target<T> {

	private final Class<T> type;
	private final String name;
	private final String url;

	@Override
	public Request apply(RequestTemplate input) {
		if (input.url().indexOf("http") != 0) {
			input.target(url());
		}
		return input.request();
	}
}
```

- ```RequestTemplate```的```input```方法
```java
public Request request() {
	if (!this.resolved) {
		throw new IllegalStateException("template has not been resolved.");
	}
	return Request.create(this.method, this.url(), this.headers(), this.requestBody());
}
```
- ```Request```的```create```方法
```java
public static Request create(HttpMethod httpMethod,
                               String url,
                               Map<String, Collection<String>> headers,
                               Body body) {
    return new Request(httpMethod, url, headers, body);
  }
```
从代码上可以看到，RequestTemplate基本上直接转为Request，没有做什么逻辑操作。

# http请求发送

SynchronousMethodHandler中构造好Request后，直接调用client的execute方法发送请求：
```java
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
	...省略部分代码
		
	response = client.execute(request, options);
	
	...省略部分代码
}
```
client是一个```Client```接口，默认实现类是```Client.Default```，使用java api中的```HttpURLConnection```发送http请求。feign还实现了：
- feign.Client.Proxied
- ApacheHttpClient
- OkHttpClient
- LoadBalancerFeignClient
- FeignBlockingLoadBalancerClient

# 接口调用过程总结
我们再将接口调用过程捋一遍：

1、接口的动态代理```Proxy```调用接口方法会执行的```FeignInvocationHandler```  
2、```FeignInvocationHandler```通过方法签名在属性```Map<Method, MethodHandler> dispatch```中找到```SynchronousMethodHandler```，调用```invoke```方法  
3、```SynchronousMethodHandler```的```invoke```方法根据传入的方法参数，通过自身属性工厂对象```RequestTemplate.Factory```创建```RequestTemplate```，工厂里面会用根据需要进行```Encode```  
4、```SynchronousMethodHandler```遍历自身属性```RequestInterceptor```列表，对```RequestTemplate```进行改造  
4、```SynchronousMethodHandler```调用自身```Target```属性的```apply```方法，将```RequestTemplate```转换为```Request```对象  
5、```SynchronousMethodHandler```调用自身```Client```的```execute```方法，传入```Request```对象  
6、```Client```将```Request```转换为http请求，发送后将http响应转换为```Response```对象  
7、```SynchronousMethodHandler```调用```Decoder```的方法对```Response```对象解码后返回  
8、返回的对象最后返回到```Proxy```  


时序图如下：  
![](https://oscimg.oschina.net/oscnet/up-5a078b81830db03301f33d1dbc860f3ef05.png)

# feign扩展点总结

前文分析源代码时，已经提到了feign的扩展点，最后我们再将feign的主要扩展点进行总结一下：

- **Contract** 契约
```Contract```的作用是解析接口方法，生成Rest定义。  
   - ```feign.Contract.Default``` feign默认实现
    - ```JAXRSContract``` javax.ws.rs注解接口实现
    - ```SpringMvcContract```是spring cloud提供SpringMVC注解实现方式。
	- ```HystrixDelegatingContract``` hyxtrix注解的实现
	
- **InvocationHandler** 动态代理handler  
通过```InvocationHandlerFactory```注入到```Feign.Builder```中，feign提供了Hystrix的扩展，实现Hystrix接入

- **Encoder** 请求body编码器
feign已经提供扩展包含：

  - 默认编码器，只能处理String和byte[]
  - json编码器```GsonEncoder```、```JacksonEncoder```
  - XML编码器```JAXBEncoder```
  - ```FormEncoder```
  - ```SpringEncoder```
  - ```SpringFormEncoder```
  - ```PageableSpringEncoder```
  
- **Decoder** http响应解码器  
最基本的有：  
  - feign默认的 ```StringDecoder```、```OptionalDecoder```、```feign.codec.Decoder.Default```、```feign.Feign.ResponseMappingDecoder```
  - json解码器 ```GsonDecoder```、```JacksonDecoder```、```JacksonIteratorDecoder```
  - XML解码器 ```JAXBDecoder```
  - Stream流解码器 ```StreamDecoder```
  - spring的 ```SpringDecoder```、```ResponseEntityDecoder```
  
- **ErrorDecoder** 错误解码器  
  
- **Target** 请求转换器  
feign提供的实现有：
  - ```HardCodedTarget``` 默认Target，不做任何处理。
  - ```EmptyTarget``` feign提供的
  
- **Client** 发送http请求的客户端  
feign提供的Client实现有：
  - ```Client.Default``` 默认实现，使用java api的```HttpClientConnection```发送http请求
  - ```ApacheHttpClient``` 使用apache的Http客户端发送请求
  - ```OkHttpClient``` 使用OKHttp客户端发送请求
  - ```LoadBalancerFeignClient``` spring-cloud-openfeign的通过负载均衡选择服务发送请求
  
- **RequestInterceptor** 请求拦截器  
调用客户端发请求前，修改```RequestTemplate```，比如为所有请求添加Header就可以用拦截器实现。
  - ```BaseRequestInterceptor```
  - ```BasicAuthRequestInterceptor```
  - ```FeignAcceptGzipEncodingInterceptor```
  - ```FeignContentGzipEncodingInterceptor```

- **Retryer** 重试策略  
默认的策略是```Retryer.Default```，包含3个参数：
  - 间隔、
  - 最大间隔
  - 重试次数，第一次失败重试前会sleep输入的间隔时间的，后面每次重试sleep时间是前一次的1.5倍，超过最大时间或者最大重试次数就失败


> 文章绝大部分内容来源于：http://techblog.ppdai.com/2018/05/14/20180514/  
> 本人在其基础上加入自己的理解