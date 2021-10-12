# 1、java se: 

## 1.1、面向对象（封装、继承、多态）、接口多继承、类单继承、类多实现多个接口， 一个java不能不能有读个public类，

private：私有，一个类里面可以访问，相同包其他类不行

public：公开访问

默认：相同包可以访问

protected：相同包、继承



## 1.2、final关键字：

- 基本数据类型：

  ```java
  // 错误
  final int a = 1; 
  a=2;
  ```

- 对象类型

  ```java
  class Student{
      string name;
      
      get;
      
      set;
  }
  
  class Test{
      public static void main(String[] args){
          // 正确
          final Student s = new Student();
          s.name = "11";
          s.name = "22"
      }
  }
  ```

  ## 1.3、static:

  ```
  class A{
  	
  	int a = 1;
  	
  	 public static void main(String[] args){
  		//不能
  		a = 2;
  		
  		// 正确
  		A aa = new A();
  		aa.a = 2;
  		
  		// 错误 
  		b();
  	}
  	
  	public void b(){
  	
  	
  	}
  }
  ```

  ## 1.4、抽象类

  `abstract`

  ```
  public abstract class Test{
  
  	
  }
  
  // 可以 
  class Test2 extends Test{
  
  	
  }
  
  class B{
  	main(){
  		// 错误
  		new Test();
  	
  	}
  }
  ```

  

  # 1.5、接口

  ```
  public interface Test{
  	// public static final
  	int a = 2;
  
  }
  
  class A{
  	public void b(){
  		// 正确
  		System.out.println(Test.a);
  		
  		// 错误
  		Test.a = 1;
  	}
  
  }
  ```

  ## 1.6、注解(annotation)

  ```
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ElementType.TYPE, ElementType.METHOD})
  public @interface RequestMapping {
  
      String value() default  "a" ;
  
      RequestMethod method() default RequestMethod.GET;
  }
  
  @RequestMapping(value = "/", method = RequestMethod.POST)
  public class A {
  
  }
  ```

  ## 1.7、反射

  ```
  package org.example;
  
  import java.lang.reflect.Field;
  
  /**
   * @author calebzhao
   * @date 2020/4/12 11:02
   */ 
  public class A {
  
      public static void main(String[] args) {
  
          B a1 = new B();
          Class<?> a1Class =  a1.getClass();
          try {
              Field a1Filed = a1Class.getDeclaredField("a");
              a1Filed.setAccessible(true);
  
              System.out.println("修改前:" + a1Filed.get(a1));
  
              a1Filed.set(a1, 2);
  
              System.out.println("修改后：");
              a1.getA();
          }
          catch (NoSuchFieldException | IllegalAccessException e) {
              e.printStackTrace();
          }
      }
  
  
      void b(){
  
      }
  }
  
  class B {
      private int a = 1;
  
      public void getA(){
          System.out.println(a);
      }
  }
  
  ```

  

  ## 1.8、集合

  ### 1.8.1、顶级接口：`List`、`Collection`、`Set` 、`Map`、`Queue`

  ### 1.8.2、List实现类： 

  - `ArrayList`

    有序的、底层使用数组存储，查询快、插入、删除慢，相同值元素可重复

  - `LinkedList`

    有序的，底层使用链表存储，查询慢、插入、删除快，相同值元素可重复

  - `Stack`

    有序的，底使用数组存储，相同值元素可重复、后进先出FIFO（first in first out）

  ### 1.8.3、Set实现类：

  `HashSet`

  无序的，元素不可重复、可添加null值，底层使用HashMap的key存储

  `LinkedHashSet`

  有序的，按插入元素顺序排序，可添加null值，底层使用HashMap的key存储

  `TreeSet`

  有序的，默认按元素的自然顺序排序，可以指定比较器Compartor来排序

  ### 1.8.4、Map实现类：

  `HashMap`

  key-value键值对，通过key查询对应的value， 元素的key、value可以为null，无序的

  `LinkedHashMap`

  key-value键值对，通过key查询对应的value，元素的key、value可以为null，有序的，默认按照插入顺序排序

  `TreeMap`

  key-value键值对，通过key查询对应的value，元素的key、value可以为null，有序的，默认按照key的自然顺序排序，可以指定比较器Compartor来自定义排序规则

  `HashTable`

  元素的key、value不能为null

  ### 1.8.5、Queue实现类：

  `Queue`

  先进先出

  `PriorityQueue`优先级队列，可以指定优先级

  `Deque` 双向队列的接口

  `LinkedList`实现了`Deque`，双向队列

  ### 1.8.6、Collections与Collection区别

  `Collections`是一个工具类，无法实例化，提供一些常用的静态方法，比如sort、binarySearch、EMPTY_LIST、EMPTY_SET、将集合转换成不可变集合、

  将集合转换成线程安全的集合

  

  Collection：集合的顶级接口

  

  ## 1.9、多线程

  实现线程的几种方式

  - 继承Thread类，调用start()方法启动线程
- 实现Runnable接口，new Thread(new XX()).start()
  
  线程的状态：
  
  新建： new Thread()
  
  可运行: 调用start()
  
  阻塞：等待获取锁、等待 系统IO完成
  
  正在运行：线程获取到CPU时间片
  
  死亡：运行结束
  
  
  
  Thread.sleep(): 睡眠，不会释放已占有的锁，会让出CPU时间片
  
  wait: 线程等待，必须先获取到锁才能调用wait方法，调用wait方法后会释放锁，会让出CPU时间片，在执行代码处停止，必须等到另一个线程调用notitify、notitifyAll方法来通知该线程唤醒，重新尝试获取锁。
  
  notitify、notitifyAll：方法来通知该线程唤醒
  
  ## 1.10、IO
  
  InputStream、OutputStream、
  
  File：文件、目录、遍历目录、重命名文件/目录、删除文件/目录、复制文件/目录、读取文件内容、修改文件内容
  
  字符流：StringReader、StringWriter、BufferedReader、BufferedWriter、FileReader、FileWriter
  
  字节流：FileInputStream、DataInputStream、BufferedInputStream、ByteArrayInputStream、ObjectInputStream
  
  
  
  ## 1.11、序列化
  
  Serializable
  
  ```java
  package org.example;
  
  import java.io.*;
  
  /**
   * @author calebzhao
   * @date 2020/4/12 15:44
   */
  public class TestSerializable {
  
      static class A implements Serializable {
          private int a = 1;
          private String b = "aa";
  
          public int getA() {
              return a;
          }
  
          public void setA(int a) {
              this.a = a;
          }
  
          public String getB() {
              return b;
          }
  
          public void setB(String b) {
              this.b = b;
          }
      }
  
      public static void main(String[] args) throws IOException, ClassNotFoundException {
          A a = new A();
          ByteArrayOutputStream baos = new ByteArrayOutputStream();
          ObjectOutputStream obs = new ObjectOutputStream(baos);
          obs.writeObject(a);
  
          ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
          ObjectInputStream ois= new ObjectInputStream(bais);
          A b = (A) ois.readObject();
          System.out.println(b.getA());
          System.out.println(b.getB());
      }
  }
  
  ```
  
  ## 1.12、反射
  
  
  
  ## 1.13、代理
  
  
  
  ## 1.14、单例模式
  
  
  
  
  
  ## 1.11、网络
  
  Socket、ServerSocket、netty
  

# 2、java web

## 2.1、servlet

- `HttpServlet`

  doGet()、doPost()、doPut()、doDelete()

- `HttpServletRequest`

  - getRequestURI();
  - getCookies();
  - getRequestDispatcher("xxx").forward(request, response);
  - getParameter("")
  - getParameterMap();
  - getMethod()
  - getHeader("")
  - getSession();
  - getContextPath()

- `HttpServletResponse`

  - sendRedirect()
  - getWriter()
  - setStatus()
  - addHeader()
  - addCookie()

## 2.2、filter

`HttpFilter`

```java
public class LoginFilter implents HttpFilter{

	private String loginUrl;
	
	public void doInit(FilterConfig config){
		loginUrl = config.getParameter("loginUrl");
	
	}

	public void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain){
		HttpSession session = request.getSession();
		User user = session.getAttribute("user");
		if(user == null){
		
			response.sendRedirect(loginUrl)
		} 
		else {
			filterChain.doFilter(request, response);
		}
	}
	
}
```

spring security   shiro

## 2.3、jsp内置对象

- application
- page
- pageContext
- request
- session
- response
- exception
- out
- config

## 2.4、jstl、el表达式(记住有这些标签即可，具体如何使用不用深究，用到的时候去查)

```
<c:if test=""></c:if>

<c:foreach items="" var="xx">
</c:foreach>

<c:set var="" value=""/>

<c:out></c:out>

<c:choose>
	<c:when test="${a not eq 1}"></c:when>
	<c:otherwise>
</choose>
```

## 2.5、http协议

- 转发

  request.getRequestDispatcher("/xxx").forward(request, response)

  1次请求

- 重定向

  response.sendRedirect("http://xxx");

  2次请求

- http状态码含义

  200 - ok

  206 - 已返回部分数据，用于分段下载

  301、302、重定向

  400 - bad request(错误的请求、不合法的请求)

  401 - 未认证

  403 - 禁止访问

  404 - Not Found，找不到资源（路径错误）

  500 - 服务端内部错误

  503 - 网关错误
  
  `HttpServletResponse`中有所有的标准状态码
  
  
  
- http协议详解：
  
  https://www.cnblogs.com/an-wen/p/11180076.html
  
  http://www.blogjava.net/zjusuyong/articles/304788.html

## 2.6、websocket

应用层协议，基于http， 用于服务端主动向客户端发送消息，该协议是全双工模式

先了解是什么东西，干什么用的，具体用法用到再学

## 2.7、listener

```java

public class MyServletContexListner implements ServletContextListener {
 
    public void contextDestroyed(ServletContextEvent arg0) {

    System.out.println("ServletContext上下文死亡");

    }
 
    public void contextInitialized(ServletContextEvent arg0) {

    	System.out.println("ServletContext上下文初始化的");

    }
}
 

```



## 2.8、cookie、session

- cookie

  用来干嘛的？

  cookie每一项参数的含义name、value、domain、expires/max-age、http、secure、path

- session

  登陆流程

  session原理：

  JSESSIONID

## 2.9、静态化freemarker



# 3、框架

- spring mvc
- spring boot
- spring security
- shiro
- hibernate
- mybatis
- redis
- rabbitmq

# 4、数据库

## 4.1、增删改查基本语法

**select、update、delete、in、inner join、left join、rIght join、where、count(*) 、group by、having、sum()、uuid()、主键、索引、外键、唯一索引、自增长主键、子查询**



用户(user): id、username、age

角色(role): id、name

菜单(menu)：id、name

用户角色表（user_role）：userId、roleId



查询所有用户：`select * from user`

查询username="张三"的用户：``select * from user where username='张三'`

查询username="张三" 或email=“aa@qq.com”的用户：``select * from user where username=? or email=?`

查询username="张三" 并且年龄在18到60岁之间的用户：``select * from user where username=? and age > 18 and age < 60  ` 

``select * from user where username=? and age between 18 and 60  ` 



查询username="张三" 或年龄在18到60岁之间的用户：

``select * from user where username=? or (age > 18 and age < 60)  ` 

查询username=“张三” 或 username=“李四” 或 username=“王五”

``select * from user where username='张三' or username='李四' or username='王五' `

``select * from user where username in('张三', '李四', '王五') `



把username="张三"的所有用户年龄改成25

```
update user set age = 25 where username = '张三'
```

删除所有用户

```
delete from user
```

查询用户张三所拥有的所有角色名称

```
select r.id, r.name from user u
inner join user_role ur on ur.userId = u.id
inner join role r on r.id = ur.roleId
where u.username = '张三'
```



## 4.2、索引

- 一般索引 
- 唯一索引
- 哈希索引
- 建立索引的策略（索引优化）
- 为什么要有索引
- explain关键字查看执行计划

## 4.3、Mysql的数据类型

**int 、bigint、tinyint、smallint、float、double、decimal、varchar、text、longtext、byte、json、char**

# 5、其他

