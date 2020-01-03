---
title: Spring中的ResolvableType详解.
date: 2020-01-02 15:51:40
tags:
- Type
- ResolvableType
- spring
- spring boot
- spring cloud
- 泛型
- 反射
categories: 
- java基础
- spring
- spring boot
---



**阅读本文之前如果对java中的Type体系不了解请先阅读我的另一篇文章：[Java中的Type详解](https://calebzhao.github.io/2019/12/29/Java中的Type详解/#反射相关接口)**



# ResolvableType

ResolvableType为所有的java类型提供了统一的数据结构以及API ，换句话说，一个ResolvableType对象就对应着一种java类型。 

我们可以通过ResolvableType对象获取类型携带的信息，举例如下：

- getSuperType()：获取直接父类型，返回ResolvableType
- getInterfaces()：获取接口类型， 返回ResolvableType[]数组
- getGeneric(int...)：获取类型携带的泛型类型，返回ResolvableType[]数组
- resolve()：Type对象到Class对象的转换

另外，ResolvableType的构造方法全部为私有的，我们不能直接new，只能使用其提供的静态方法进行类型获取：
- forField(Field)：获取指定字段的类型
- forMethodParameter(Method, int)：获取指定方法的指定形参的类型
- forMethodReturnType(Method)：获取指定方法的返回值的类型
- forClass(Class)：直接封装指定的类型


# spring 中处理泛型类

```java
package com.calebzhao.test;

import org.springframework.core.ResolvableType;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.Arrays;
import java.util.Collection;
import java.util.stream.IntStream;

public class SpringTypeTest {

    /**
     * 测试继承获取泛型
     * @param <T>
     */
    class Parent<T, S>{

    }

    interface IParent1<T> {

    }

    interface IParent2<T> {

    }

    public class Children<K extends String, V extends Collection> extends Parent<String, Boolean> implements IParent1<Long>, IParent2<Double> {

    }

    public static void main(String[] args) {
        System.out.println("------------------------jdk原生方式-获取泛型父类---------------------------------------");
        // jdk原生方式获取Children类继承的父类的类型
        Type genericSuperclassType = Children.class.getGenericSuperclass();
        // com.calebzhao.test.SpringTypeTest$Parent<java.lang.String, java.lang.Boolean>
        System.out.println(genericSuperclassType);

        if (genericSuperclassType instanceof ParameterizedType) {
            Type[] actualTypeArguments = ((ParameterizedType) genericSuperclassType)
                    .getActualTypeArguments();
            for (Type argumentType : actualTypeArguments) {
                System.out.println("父类ParameterizedType.getActualTypeArguments:" + argumentType);
            }
        }

        System.out.println("------------------------jdk原生方式-获取泛型父接口---------------------------------------");


        // jdk原生方式获取Children类实现的接口的类型
        Type[] genericInterfacesTypes = Children.class.getGenericInterfaces();
        // [com.calebzhao.test.SpringTypeTest$IParent1<java.lang.Long>, com.calebzhao.test.SpringTypeTest$IParent2<java.lang.Double>]
        System.out.println(Arrays.toString(genericInterfacesTypes));

        for (Type interfaceType : genericInterfacesTypes) {
            if (interfaceType instanceof ParameterizedType) {
                Type[] actualTypeArguments = ((ParameterizedType) interfaceType)
                        .getActualTypeArguments();
                for (Type argumentType : actualTypeArguments) {
                    System.out.println("父接口ParameterizedType.getActualTypeArguments:" + argumentType);
                }
            }
        }


        System.out.println("------------------------分割线 spring ResolvableType---------------------------------------");
        ResolvableType childrenResolvableType = ResolvableType.forClass(Children.class);
        System.out.println("children type：" + childrenResolvableType.getType());
        System.out.println("children raw type：" + childrenResolvableType.getRawClass());
        System.out.println("children generics：" + Arrays.toString(childrenResolvableType.getGenerics()));

        System.out.println("-----super ResolvableType-------");
        ResolvableType superResolvableType = childrenResolvableType.getSuperType();
        System.out.println("super generics：" + Arrays.toString(superResolvableType.getGenerics()));
        System.out.println("super type：" + superResolvableType.getType());
        System.out.println("super raw class：" + superResolvableType.getRawClass());
        System.out.println("super getComponentType：" +  superResolvableType.getComponentType());
        System.out.println("super getSource：" +  superResolvableType.getSource());
        System.out.println("super：" + Arrays.toString(superResolvableType.getInterfaces()));

        System.out.println("\n-----interface ResolvableType-------");
        ResolvableType[] interfaceResolvableTypes = childrenResolvableType.getInterfaces();
        IntStream.range(0, interfaceResolvableTypes.length).forEach(index ->{
            ResolvableType interfaceResolvableType = interfaceResolvableTypes[index];

            System.out.println("\n -------第" + index + "个接口------------");
            System.out.println("interface generics：" + Arrays.toString(interfaceResolvableType.getGenerics()));
            System.out.println("interface type：" + interfaceResolvableType.getType());
            System.out.println("interface raw class：" + interfaceResolvableType.getRawClass());
            System.out.println("interface getComponentType：" +  interfaceResolvableType.getComponentType());
            System.out.println("interface getSource：" +  interfaceResolvableType.getSource());
            System.out.println("interface：" + Arrays.toString(interfaceResolvableType.getInterfaces()));
        });
    }
}

```

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200103160441.png)

# spring 中处理泛型参数

```java
package com.calebzhao.test;

import org.springframework.core.ResolvableType;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.IntStream;

public class SpringResolvableTypeGenericClass {

    // 父类
    class Parent<T, S>{

    }

    // 父接口1
    interface IParent1<T> {

    }

    // 父接口2
    interface IParent2<T> {

    }

    // 子类， 继承父类Parent， 实现接口IParent1，IParent2
    public class Children<K extends String, V extends Collection>
            extends Parent<String, Boolean>
            implements IParent1<Long>, IParent2<Double> {


    }

    private List<String> listString;

    private List<List<String>> listLists;

    private Map<String, Long> maps;

    private Parent<String, Double> parent;



    public Map<String, Long> getMaps() {
        return maps;
    }

    /**
     *  private Parent<String, Double> parent;
     *
     */
    public static void  doTestFindParent(){
        System.out.println("-----------private Parent<String, Double> parent;字段----------------\n");
        // 通过反射找到   private Parent<String, Double> parent;字段
        Field parentField = ReflectionUtils.findField(SpringResolvableTypeGenericClass.class,"parent");

        // 获取parent字段的ResolvableType
        ResolvableType parentResolvableType = ResolvableType.forField(parentField);
        System.out.println("parent属性的类型："+parentResolvableType.getType());

        //获取第0个位置的参数泛型
        ResolvableType[] generics = parentResolvableType.getGenerics();
        IntStream.range(0, generics.length).forEach(index -> {
            ResolvableType resolvableType = generics[index];

            Class<?> resolve = resolvableType.resolve();
            System.out.println("parent属性的第"+index+"个泛型参数："+resolve);
        });

//        -----------private Parent<String, Double> parent;字段----------------
//
//        parent属性的类型：com.calebzhao.test.SpringResolvableTypeGenericClass$Parent<java.lang.String, java.lang.Double>
//        parent属性的第0个泛型参数：class java.lang.String
//        parent属性的第1个泛型参数：class java.lang.Double
    }

    /**
     * private List<String> listString;
     */
    public static void  doTestFindListStr(){
        System.out.println("\n-----------private List<String> listString;字段----------------\n");
        // 通过反射找到  private List<String> listString;字段
        Field listStringField = ReflectionUtils.findField(SpringResolvableTypeGenericClass.class,"listString");

        // 获取listString字段的ResolvableType
        ResolvableType listStringResolvableType = ResolvableType.forField(listStringField);
        System.out.println("listString属性类型："+listStringResolvableType.getType());

        //获取第0个位置的参数泛型
        Class<?> resolve = listStringResolvableType.getGeneric(0).resolve();
        System.out.println("listString属性泛型参数："+resolve);

//        -----------private List<String> listString;字段----------------
//
//        listString属性类型：java.util.List<java.lang.String>
//        listString属性泛型参数：class java.lang.String
    }

    /**
     * private List<List<String>> listLists;
     * listLists type:java.util.List<java.util.List<java.lang.String>>
     * 泛型参数为：interface java.util.List
     * 泛型参数为：class java.lang.String
     * 泛型参数为：class java.lang.String
     * begin 遍历
     * 泛型参数为：java.util.List<java.lang.String>
     * end 遍历
     */
    public static void  doTestFindlistLists(){
        System.out.println("\n-----------private List<List<String>> listLists;字段----------------\n");
        // 通过反射找到   private List<List<String>> listLists;字段
        Field  listListsField = ReflectionUtils.findField(SpringResolvableTypeGenericClass.class,"listLists");

        // 获取listLists字段的ResolvableType
        ResolvableType listListsResolvableType = ResolvableType.forField(listListsField);
        System.out.println("listLists属性的类型："+listListsResolvableType.getType());

        //获取第0个位置的参数泛型
        Class<?> resolve = listListsResolvableType.getGeneric(0).resolve();
        System.out.println("listLists属性的泛型参数："+resolve);

        //region 访问嵌套泛型，泛型参数为：class java.lang.String
        resolve = listListsResolvableType.getGeneric(0).getGeneric(0).resolve();
        System.out.println("listLists属性的嵌套泛型参数为："+resolve);

        // 和上面的代码等价
        resolve = listListsResolvableType.getGeneric(0,0).resolve();
        System.out.println("泛型参数为："+resolve);
        //end region

        ResolvableType[] resolvableTypes = listListsResolvableType.getGenerics();
        System.out.println("begin 遍历");
        for(ResolvableType resolvableType: resolvableTypes){
            resolve = resolvableType.resolve();
            System.out.println("泛型参数为："+resolve);
        }
        System.out.println("end 遍历");

//        -----------private List<List<String>> listLists;字段----------------
//
//        listLists属性的类型：java.util.List<java.util.List<java.lang.String>>
//        listLists属性的泛型参数：interface java.util.List
//        泛型参数为：class java.lang.String
//        泛型参数为：class java.lang.String
//        begin 遍历
//        泛型参数为：interface java.util.List
//        end 遍历
    }


    /**
     *  private Map<String, Long> maps;
     */
    public static void  doTestFindMaps(){
        System.out.println("\n-----------private Map<String, Long> maps;字段----------------\n");
        // 通过反射找到 private Map<String, Long> maps;字段
        Field mapsField = ReflectionUtils.findField(SpringResolvableTypeGenericClass.class,"maps");

        // 获取maps字段的ResolvableType
        ResolvableType mapsResolvableType = ResolvableType.forField(mapsField);
        System.out.println("maps属性类型:"+mapsResolvableType.getType());

        System.out.println("begin 遍历");
        ResolvableType[] resolvableTypes = mapsResolvableType.getGenerics();
        Class<?> resolve =null;
        for(ResolvableType resolvableType: resolvableTypes){
            resolve = resolvableType.resolve();
            System.out.println("泛型参数为："+resolve);
        }
        System.out.println("end 遍历");

//        -----------private Map<String, Long> maps;字段----------------
//
//        maps属性类型:java.util.Map<java.lang.String, java.lang.Long>
//        begin 遍历
//        泛型参数为：class java.lang.String
//        泛型参数为：class java.lang.Long
//        end 遍历

    }

    /**
     * Map<String, Long>
     */
    public static void doTestFindReturn(){
        System.out.println("\n-----------public Map<String, Long> getMaps(){ ... } 方法----------------\n");
        // 通过反射找到 public Map<String, Long> getMaps(){ ... } 方法
        Method getMapsMethod = ReflectionUtils.findMethod(SpringResolvableTypeGenericClass.class, "getMaps");

        // 获取getMaps方法的返回值的ResolvableType
        ResolvableType methodReturnTypeResolvableType = ResolvableType.forMethodReturnType(getMapsMethod);

        System.out.println("getMaps方法的返回值的类型："+methodReturnTypeResolvableType.getType());

        System.out.println("begin 遍历");
        ResolvableType[] resolvableTypes = methodReturnTypeResolvableType.getGenerics();
        Class<?> resolve =null;
        for(ResolvableType resolvableTypeItem: resolvableTypes){
            resolve = resolvableTypeItem.resolve();
            System.out.println("泛型参数为："+resolve);
        }
        System.out.println("end 遍历");

//        -----------public Map<String, Long> getMaps(){ ... } 方法----------------
//
//        getMaps方法的返回值的类型：java.util.Map<java.lang.String, java.lang.Long>
//        begin 遍历
//        泛型参数为：class java.lang.String
//        泛型参数为：class java.lang.Long
//        end 遍历
    }

    /**
     *
     * @param args
     */
    public static void main(String[] args) {
        doTestFindParent();
        doTestFindListStr();
        doTestFindlistLists();
        doTestFindMaps();
        doTestFindReturn();
    }
}

```





# 总结

总结一句话就是使用起来非常的简单方便，更多超级复杂的可以参考spring 源码中的测试用例：`ResolvableTypeTests`，其实这些的使用都是在Java的基础上进行使用的哦！

**Type是Java 编程语言中所有类型的公共高级接口**（官方解释），也就是Java中所有类型的“爹”；其中，“所有类型”的描述尤为值得关注。它并不是我们平常工作中经常使用的 int、String、List、Map等数据类型，而是从Java语言角度来说，**对基本类型、引用类型向上的抽象**；

**Type体系中类型的包括**：**原始类型(Class)**、**参数化类型(ParameterizedType)**、**数组类型(GenericArrayType)**、**类型变量(TypeVariable)**、**基本类型(Class)**

- 原始类型，不仅仅包含我们平常所指的类，还包括枚举、数组、注解等；
- 参数化类型，就是我们平常所用到的泛型List、Map；
- 数组类型，并不是我们工作中所使用的数组String[] 、byte[]，而是带有泛型的数组，即T[] ；
- 基本类型，也就是我们所说的java的基本类型，即int,float,double等