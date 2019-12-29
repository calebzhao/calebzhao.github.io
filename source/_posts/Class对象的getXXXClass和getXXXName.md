---
title: Class对象的getXXXClass和getXXXName
date: 2019-12-29 16:51:55
tags: java
categories: java基础
---

# getXXXClass方法获取class对象
在讲解具体的方法之前，我们先介绍一下java类（接口）的划分方法。
java的class对象分为5种，这一点在getEnclosingClass方法的注释中有写明，分别是：
- a. Top类
- b. Nested类：嵌套类，即静态成员类
- c. Inner类：内部类，即普通成员类
- d. Local类：局部类，在方法中定义的类
- e. Anonymous：匿名类

Class对象有3个和name相关的方法，分别是```getClass```, ```getDeclaringClass```和```getEnclosingClass```，下面我们仔细看一下这3个方法有什么区别。

## getClass方法
```getClass```方法是从```Object```类继承而来的方法，返回的是该对象所对应的运行时类。
```class.getClass()```返回的自然是```java.lang.Class```。

## getDeclaringClass方法
从注释可以看出，```class.getDeclaringClass()```方法返回了声明class对象的类，只有当class对象的类A是另一个类B的成员类，该方法才能返回值，返回该成员类所在的类（即类B），否则返回null值。

如果class对象是local class，是在类B的方法里定义的，同样返回null值，而不会返回类B。
```java
/**
 * If the class or interface represented by this {@code Class} object
 * is a member of another class, returns the {@code Class} object
 * representing the class in which it was declared.  This method returns
 * null if this class or interface is not a member of any other class.  If
 * this {@code Class} object represents an array class, a primitive
 * type, or void,then this method returns null.
 * @return the declaring class for this class
 * @throws SecurityException
 *         If a security manager, <i>s</i>, is present and the caller's
 *         class loader is not the same as or an ancestor of the class
 *         loader for the declaring class and invocation of {@link
 *         SecurityManager#checkPackageAccess s.checkPackageAccess()}
 *         denies access to the package of the declaring class
 * @since JDK1.1
   */
@CallerSensitive
public Class<?> getDeclaringClass() throws SecurityException {
	final Class<?> candidate = getDeclaringClass0();

	if (candidate != null)
		candidate.checkPackageAccess(
		ClassLoader.getClassLoader(Reflection.getCallerClass()), true);
	return candidate;
}
```

## getEnclosingClass方法

返回该class对象的立即封闭类，即该类实在哪个类里定义的，与getDeclaringClass不同的是getEnclosingClass并不要求该class一定是成员类。
```java
@CallerSensitive
public Class<?> getEnclosingClass() throws SecurityException {
	EnclosingMethodInfo enclosingInfo = getEnclosingMethodInfo();
	Class<?> enclosingCandidate;

	if (enclosingInfo == null) {
		// This is a top level or a nested class or an inner class (a, b, or c)
		enclosingCandidate = getDeclaringClass();
	} else {
		Class<?> enclosingClass = enclosingInfo.getEnclosingClass();
		// This is a local class or an anonymous class (d or e)
		if (enclosingClass == this || enclosingClass == null)
			throw new InternalError("Malformed enclosing method information");
		else
			enclosingCandidate = enclosingClass;
	}

	if (enclosingCandidate != null)
		enclosingCandidate.checkPackageAccess(
		ClassLoader.getClassLoader(Reflection.getCallerClass()), true);
	return enclosingCandidate;
}
```

总结一下：  
![](https://oscimg.oschina.net/oscnet/up-4641e002b6ee33a316b5620a4afc0bb6f02.png)

示例代码：
```java
package com.calebzhao.openfeign;

import org.junit.Test;

import java.util.Arrays;

public class ClassTest {

	private Object obj;

	static class A {

	}

	class B {

	}

	interface C{
		void run();
	}

	public void method1(){
		Class cClass = new C(){
			@Override
			public void run() {
				System.out.println("run");
			}
		}.getClass();

		System.out.println(cClass.getClass());
		System.out.println(cClass.getEnclosingClass());
		System.out.println(cClass.getDeclaringClass());
		System.out.println(Arrays.toString(cClass.getDeclaredClasses()));
	}

	public void method2(){
		class D{

		}

		Class d = D.class;
		System.out.println(d.getClass());
		System.out.println(d.getEnclosingClass());
		System.out.println(d.getDeclaringClass());
		System.out.println(Arrays.toString(d.getDeclaredClasses()));
	}

	@Test
	public void testClassTest() {
		System.out.println("*************ClassTest*******************\n");
		Class<? extends ClassTest> testClass = ClassTest.class;
		System.out.println(testClass.getClass());
		System.out.println(testClass.getEnclosingClass());
		System.out.println(testClass.getDeclaringClass());
		System.out.println(Arrays.toString(testClass.getDeclaredClasses()));


		System.out.println("\n*************A*******************\n");
		Class<? extends A> aClass = A.class;
		System.out.println(aClass.getClass());
		System.out.println(aClass.getEnclosingClass());
		System.out.println(aClass.getDeclaringClass());
		System.out.println(Arrays.toString(aClass.getDeclaredClasses()));


		System.out.println("\n*************B*******************\n");
		Class<? extends B> bClass = B.class;
		System.out.println(bClass.getClass());
		System.out.println(bClass.getEnclosingClass());
		System.out.println(bClass.getDeclaringClass());
		System.out.println(Arrays.toString(bClass.getDeclaredClasses()));

		System.out.println("\n*************C*******************\n");
		method1();

		System.out.println("\n*************D***********\n");

		method2();
	}

}
```

输出结果如下:
```
*************ClassTest*******************

class java.lang.Class
null
null
[interface com.calebzhao.openfeign.ClassTest$C, class com.calebzhao.openfeign.ClassTest$B, class com.calebzhao.openfeign.ClassTest$A]

*************A*******************

class java.lang.Class
class com.calebzhao.openfeign.ClassTest
class com.calebzhao.openfeign.ClassTest
[]

*************B*******************

class java.lang.Class
class com.calebzhao.openfeign.ClassTest
class com.calebzhao.openfeign.ClassTest
[]

*************C*******************

class java.lang.Class
class com.calebzhao.openfeign.ClassTest
null
[]

*************D***********

class java.lang.Class
class com.calebzhao.openfeign.ClassTest
null
[]



进程已结束，退出代码 0
```

# getXXXName方法返回名称

在讲具体的区别之前，先明确对象的描述符概念。
在class文件里，字段表集合里存储了类的属性和方法，并且通过描述符来区分不同的对象类型。对于基本数据结构，描述符用一个大写字母表示；对于对象，描述符用L+类的全限定名表示，并以分号结束。
描述符在class类中getName方法的注释中也有描述。

![](https://oscimg.oschina.net/oscnet/up-6bbdad6f895c55e46eed327943ec480890c.png)

Class对象有3个和name相关的方法，分别是getSimpleName, getName和getCanonicalName，下面我们仔细看一下这3个方法有什么区别。

## getName
在class类里，```getName```方法通过调用native方法```getName0```获得，返回class对象在虚拟机里面的表示。
1. 对于top class，```getName```为:类的全限定名，如：```test.Test```。
2. 对于nested class和inner class，```getName```为：```getEnclosingClass```的```getName``` + $ + 类名，如```test.Test$Inner```。
3.  对于anonymous class，getName为：```getEnclosingClass```的```getName``` + $ + 虚拟机分配的类名（虚拟机对于每一个类中的匿名类会按顺序分配给1，2之类的类名，如```test.Test$1```。

    对于local class，```getName```为：```getEnclosingClass```的```getName``` + $ + 虚拟机分配的前缀+类名（由于local class是属于方法的，一个类中的不同方法下的local class类名是有可能相同的，所以虚拟机会在类名前面自动加上按方法加上1，2之类的排序），如```test.Test$2LocalClass```。

4. 对于基本数据结构的class对象，```getName```为：基本数据结构的名称，如```boolean```。
5. 对于数组的class对象，```getName```为：```[ + 元素类型的描述符```，如```[Ljava.lang.String;(带分号)，[Z```。如果是多维数组，则带有多个[。

## getSimpleName方法

从代码可以看出
1. 对于数组的class对象，返回元素类型的```getSimpleName+[]```。
2. 对于非top class的对象，获取```getEnclosingClass```，从```getName```中截去```getEnclosingClass```的```getName```，截去```$```符号，然后截去字符串前面的所有数字。
3. 对于top class的对象，返回类名。

```java
public String getSimpleName() {
	if (isArray())
		return getComponentType().getSimpleName()+"[]";

	String simpleName = getSimpleBinaryName();
	if (simpleName == null) { // top level class
		simpleName = getName();
		return simpleName.substring(simpleName.lastIndexOf(".")+1); // strip the package name
	}
	int length = simpleName.length();
	if (length < 1 || simpleName.charAt(0) != '$')
		throw new InternalError("Malformed class name");
	int index = 1;
	while (index < length && isAsciiDigit(simpleName.charAt(index)))
		index++;
	// Eventually, this is the empty string iff this is an anonymous class
	return simpleName.substring(index);
}
```

```java
private String getSimpleBinaryName() {
	Class<?> enclosingClass = getEnclosingClass();
	if (enclosingClass == null) // top level class
		return null;
	// Otherwise, strip the enclosing class' name
	try {
		return getName().substring(enclosingClass.getName().length());
	} catch (IndexOutOfBoundsException ex) {
		throw new InternalError("Malformed class name", ex);
	}
}
```

## getCanonicalName方法

1. 对于数组的class对象，检查元素类型的```getCanonicalName```，如果是null，则返回null；否则元素类型的```getCanonicalName + []```。
2. 对于local class和anonymous class，返回null。
3. 对于top class，返回```getName```。
4. 获取```getEnclosingClass```的```getCanonicalName```作为canonicalName，为null则返回null；否则getCanonicalName + . + getSimpleName。

```java
public String getCanonicalName() {
	if (isArray()) {
		String canonicalName = getComponentType().getCanonicalName();
		if (canonicalName != null)
			return canonicalName + "[]";
		else
			return null;
	}
	if (isLocalOrAnonymousClass())
		return null;
	Class<?> enclosingClass = getEnclosingClass();
	if (enclosingClass == null) { // top level class
		return getName();
	} else {
		String enclosingName = enclosingClass.getCanonicalName();
		if (enclosingName == null)
			return null;
		return enclosingName + "." + getSimpleName();
	}
}
```

总结一下：

![](https://oscimg.oschina.net/oscnet/up-68a74ffa2f2229c5ca1f6ea7bad6b3ccc32.png)

# spring包里的classMetadata

在spring的```org.springframework.core.type```包下的```ClassMetadata```接口里有一些与我们上述内容相关的方法，这些方法在spring的源码中经常会遇到，我们通过其实现类```StandardClassMetadata```一并解释一下：

- isIndependent

从方法名称上可以看出，判断对应的类是否是独立的，不需要依赖其它类。
如果是top class或nested class，视为Independent，返回true；否则返回false。
```java
@Override
public boolean isIndependent() {
	return (!hasEnclosingClass() ||
			(this.introspectedClass.getDeclaringClass() != null &&
			Modifier.isStatic(this.introspectedClass.getModifiers())));
}
```

- hasEnclosingClass

从方法名称上可以看出，判断getEnclosingClass是否存在。
如果是top class返回false，否则返回true。

```java
@Override
public boolean hasEnclosingClass() {
	return (this.introspectedClass.getEnclosingClass() != null);
}
```

- getEnclosingClassName

从方法名称上可以看出，获取```getEnclosingClass```的getName。
如果是top class返回null，否则返回```getEnclosingClass```的getName。
```java
@Override
@Nullable
public String getEnclosingClassName() {
	Class<?> enclosingClass = this.introspectedClass.getEnclosingClass();
	return (enclosingClass != null ? enclosingClass.getName() : null);
}
```

- getClassName

从方法名称上可以看出对于class对象的```getName```。
```java
@Override
public String getClassName() {
	return this.introspectedClass.getName();
}
```