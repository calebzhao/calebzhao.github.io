---
title: 深入理解jvm
date: 2019-12-30 08:22:37
tags:
- java
- jvm
categories: java基础
---

# 1、前置知识
## 1.1、jvm参数使用形式
```-XX:+&lt;option&gt;``` 表示开启option选项

```-XX:-&lt;option&gt;```  表示关闭option选项

```-XX:&lt;option&gt;=&lt;value&gt;``` 表示option选项的值设置为value  

**注意上述符号：X是大写的，“-”是减号，“+”是加号**

## 1.2、查看编译后的.class文件的字节码指令
进入到class文件所在目录，执行下述命令：

javap com.mysql.Driver 查看反编译后的源代码

**javap -c xxx.class**

javap -v **xxx.class 查看详细的字节码信息**

## 1.3、追踪jvm加载与卸载class
-XX:+TraceClassLoading，用于追踪类的加载信息并打印出来

-XX:+TraceClassUnloading 跟踪类的卸载

![图片](https://uploader.shimo.im/f/J1RoTRiIkcoNGu5m.png!thumbnail)

打印如下：

![图片](https://uploader.shimo.im/f/mFwniEDrKBcBdWpM.png!thumbnail)

## **1.4、jvm指令**
```
// ldc指令表示将int、float、String类型的常量值从常量池推送至栈顶
public static final String salary = "hello world";
// bipush指令表示将单字节（-128到127）的常量池推送至栈顶
public static final short s = 7;

// bipush指令将除了-1 到 5 外的int类型t从常量池推送至栈顶
public static final int i2_ = -2
// iconst_m1指令将-1从常量池推送至栈顶
public static final int i1_ = -1;
// iconst_0指令将0从常量池推送至栈顶
public static final int i0 = 0;
// iconst_1指令将1从常量池推送至栈顶
public static final int i1 = 1;
// iconst_2指令将2从常量池推送至栈顶
public static final int i2 = 2;
// iconst_3指令将3从常量池推送至栈顶
public static final i = 3;

// iconst_4指令将4从常量池推送至栈顶
public static final int i = 4;

// iconst_5指令将5从常量池推送至栈顶
public static final int i5= 5;
// bipush指令将除了-1 到 5 外的int类型t从常量池推送至栈顶
public static final int i6= 6;
// iconst_1
public static final boolean b1 = true;

// iconst_1
public static final byte byte1 = 1;
// bipush

public static final char c1 = 'A';
```



# 2、类的加载过程及ClassLoader
## 2.1、类加载的概念
### 2.1.1、类加载的概念
类的加载是指将类的.class文件中的二进制数据读入内存中，将其放在运行时数据区的**方法区**中，然后在**内存中创建一个java.lang.Class对象**（**规范并未说明Class对象放在哪里，HotSpot虚拟机将其放在了方法区中**）用来封装类在方法区中的数据结构。

### 2.2.2、加载.class文件的方式
* 从本地文件系统中直接加载
* 通过网络下载.class文件
* 从zip、jar登归档文件中加载c.lass文件
* 从专有数据库中提取.class文件
* 将java源文件动态编译为.class文件（应用场景：动态代理、asm、cglib、jsp编译为servlet等）
## 2.2、类加载的过程
### 2.2.1、类加载的5大阶段
![图片](https://uploader.shimo.im/f/ztaaU35sN8c7kqpV.png!thumbnail)

![图片](https://uploader.shimo.im/f/BarJeBWM4YsfZcm5.png!thumbnail)

1. 加载阶段：加载字节码

2. 链接阶段：

   2.1. 验证阶段：验证字节码是否正确
   
   2.1.1. 魔数是否正确、
   2.1.2. 字节码的主从版本号是否能被当前虚拟机执行
   ​2.1.3. 类定义是否正确，如类实现了接口，则素有方法必须都实现，抽象类里的没有实现的方法是否是抽象方法，final不能被改
   
    2.2. 准备阶段：为静态变量分配内存空间，并**赋****默****认****值**（**注意这里并不是赋初始值**）
   
    2.3. 解析阶段
   
3. 初始化阶段：为类的静态变量**赋初始值**(**静态代码块的执行也是在这一步**)
    类的初始化过程, 这几个阶段是**按顺序开始，而不是按顺序进行或完成**，

 因为这些阶段通常都是**互相交叉地混合进行的**，通常在一个阶段执行的过程中调用或激活另一个阶段。

# 2.3、导致类初始化的jvm规范
类初始化是类加载过程的最后一个阶段，到初始化阶段，才真正开始执行类中的Java程序代码。虚拟机规范严格规定了有且只有5种情况必须立即对类进行初始化：

>**第一种**：遇到new、getstatic、putstatic、invokestatic这四条字节码指令时，如果类还没有进行过初始化，则需要先触发其初始化。生成这四条指令最常见的Java代码场景是：使用new关键字实例化对象时、读取或设置一个类的静态字段（static）时（被static修饰又被final修饰的，**已在编译期把结果放入常量池的静态字段除外**）、以及调用一个类的静态方法时。
>**第二种**：使用Java.lang.refect包的方法对类进行反射调用时，如果类还没有进行过初始化，则需要先触发其初始化。
>**第三种**：**初始化一个类的子类**，当初始化一个类的时候，如果发现其父类还没有进行初始化，则需要先触发其父类的初始化。
>第四种：当虚拟机启动时，用户需要指定一个要执行的主类，虚拟机会先执行该主类。
>第五种：当使用JDK1.5支持时，如果一个java.langl.incoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

虚拟机规定**有且只有**这5种情况才会**触发类的初始化**，这5中场景中的行为称为对一个类进行**主动引用**，除此之外所有引用类的方式都不会触发其**初始化阶段**（注意这里没有说不加载这个类），称为**被动引用**。下面举一些例子来说明主动使用和被动引用。

## 2.4、导致类初始化的代码示例
### 2.4.1、通过子类引用父类中的静态字段，这时对子类的引用为被动引用，因此不会初始化子类（可能会加载子类），只会初始化父类
**类加载器并不需要等待某个类被“首次使用”时再加载它**

jvm添加参数： *-XX:+TraceClassLoading 用于追踪类的加载信息并打印出来*

```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
        // 重点：重点注意，父类会初始化，但是不会导致子类初始化
        System.out.println(Child.salary);
    }
}

class Parent{
    public static int salary = 100;


    static {
        System.out.println("Obj 初始化");
    }

    static void print(){
        System.out.println(salary);
    }
}


class Child extends Parent{

    public static int age = 32;

    static {
        System.out.println("Child 初始化");
    }
}
```

![图片](https://uploader.shimo.im/f/VFbmJHJbTh8cweZo.png!thumbnail)

从输出可以看出这里**并没有使用到Child子类**, 但是**jvm加载了父类Obj, 也加载了子类Child**，**初始化了父类Obj, 但是并没有初始化子类，**验证了前面所说的**“****类加载器并不需要等待某个类被“首次使用”时再加载它****”**

### 2.4.2、子类继承父类，主动引用子类时，先初始化父类，再初始化子类（并不适用于接口）
```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
       System.out.println(Child.age);
    }
}


class Parent{
    public static int salary = 100;

    static {
        System.out.println("Obj 初始化");
    }

    public static void print(){
        System.out.println(salary);
    }
}

class Child extends Parent{
    public static int age = 32;
    static {
        System.out.println("Child 初始化");
    }
}
```
![图片](https://uploader.shimo.im/f/Z04M2ueV14YQTKo9.png!thumbnail)

### 2.4.3、初始化一个类时并不会初始化其父接口，但是却会加载父接口
```
interface ParentInterface{

    int a = 1;

    int b = new Random().nextInt(10);

    Thread c = new Thread(){
        {
            System.out.println("ParentInterface");
        }
    };
}

class ChildClass implements ParentInterface{
    public static int b = 2;
}

public class Demo{
  public static void main(String[] args)   {
      System.out.println(ChildClass.b);
  }
}
```
输出结果：

![图片](https://uploader.shimo.im/f/IqOQSw2MblwOIsMX.png!thumbnail)

可以看到加载了ParentInterface.class， 但是并没有输出“ParentInterface”, 验证了上面说的子类实现接口时，主动引用子类不会导致父接口的初始化， 如果子类引用的变量被final修饰了，那么子类也不会加载

### 2.4.3、编译器可确定的静态常量不会导致类的加载及初始化(注意接口中的变量都是static final)
**常量在编译阶段会存入到调用这个常量的方法所在的类的常量池中**，本质上，调用类并没有直接引用到定义常量的类，因此并不会触发定义常量的类的初始化。

注意：这里指的是将常量存放到了ClassInitialDemo的常量池中，之后ClassInitialDemo与FinalObj就没有任何关系了，**删除了FinalObj.class文件，ClassInitialDemo仍然能正常运行**

```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
       // 重点：编译器可确定的静态常量不会导致类初始化
       System.out.println(FinalObj.salary);
       System.out.println(FinalObj.s);

    }
}

class FinalObj{
    // ldc指令，对于确定值得常量在编译器把他放到常量池了，不会导致类的加载及初始化
    // 这里如果删除final会导致类的加载初始化
    public static final String salary = "hello world";
    
    // bipush指令
    public static final short s = 7;


    static {
        System.out.println("FinalObj 初始化");
    }
}


```
**颠覆三观****：运行前删除FinalObj.class文件，但是代码却能正常运行**

![图片](https://uploader.shimo.im/f/ZD4atx9RHLYBfc14.png!thumbnail)

通过命令行执行javap -c ClassInitialDemo.class查看字节码

![图片](https://uploader.shimo.im/f/qPysGGdvAWErbV0n.png!thumbnail)

**可以看到在ClassInitialDemo中已经存在hello world, **

### **2.4.4、接口中变量的加载**
**结论：**接口中的变量默认都是public statc final的，所以编译期就会把a和b放到其运行类所在的常量池中，运行时和ParentInterface.class和ChildInterface.class已经没有任何关系了。

```
interface ParentInterface{

    int a = 1;

    int b = new Random().nextInt(10);
}


interface ChildInterface extends ParentInterface{
    int b = 2;
}
```



测试代码

```
System.out.println(ParentInterface.a);
System.out.println(ChildInterface.a);
System.out.println(ChildInterface.b);
System.out.println(ChildInterface.d);
```


编译后先删除ParentInterface.class和ChildInterface.class, 然后再运行上述代码会发现这2个类不存在了，仍然能正常运行，通过 jvm追加参数*-XX:+TraceClassLoading*打印的日志会发现也没有加载这2个类。

执行结果：

![图片](https://uploader.shimo.im/f/FqtGRpDlp0Ush3jj.png!thumbnail)

### 2.4.5、初始化子接口时并不会初始化其父接口
```
interface ParentInterface{
    Thread c = new Thread(){
        {
            System.out.println("ParentInterface");
        }
    };
}
interface ChildInterface extends ParentInterface{
    Thread d = new Thread(){
        {
            System.out.println("ChildInterface");
        }
    };
}

}


public class Demo{
  public static void main(String[] args)   {

      System.out.println(ChildInterface.d);
  ｝
｝
```

输出结果:
![图片](https://uploader.shimo.im/f/hePuTj91OS04PpfZ.png!thumbnail)

### 从输出可以看到父接口和子接口都被加载了，但是只初始化了子接口，而没有初始化父接口
### 2.4.6、编译期不能确定的静态常量会导致类初始化
```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
       // 重点：编译器不能确定的静态常量会导致类初始化
       System.out.println(FinalObj.x);
    }
}

class FinalObj{
    //虽然被final修饰，但是其值在编译器不能确定，必须要类初始化后才能确定，所以会导致类的初始化
    public static final int x = new Random().nextInt(10);

    static {
        System.out.println("FinalObj 初始化");
    }
}
```

![图片](https://uploader.shimo.im/f/m1uEbAAdRMYoIISH.png!thumbnail)

### 2.4.7、static初始化执行顺序是在所有静态变量赋完默认值后再执行静态变量初始化及静态代码块的
```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
     
       ObjInitial2.getInstance().print();
    }
}

/**
 * 注意
 * 初始化顺序如下：
 * 1、x = 0 静态变量赋默认值
 * 2、y = 0 静态变量赋默认值
 * 3、instance = null 静态变量赋默认值null
 * 4、z = 0 静态变量赋默认值
 * 5、x = 0 静态变量初始化
 * 5、instance = new ObjInitial2() 静态变量初始化导致构造函数初始化 执行x++ y++
 * 6、instance 之后的第一个static静态代码块初始化， 执行x++ y++
 * 7、z = x+1初始化
 * 8、执行z之后的静态代码块
 */

class ObjInitial2{
    private static int x = 0;
    private static int y;

    private static ObjInitial2 instance = new ObjInitial2();
    
    static {
        x++;
        y++;
        System.out.println("ObjInitial2 static 初始化");
    }
    
    public static int z = x + 1;
    
    static {
        System.out.println("z=" + z);
    }


    public ObjInitial2(){
        x++;
        y++;
        System.out.println("ObjInitial2 new 初始化");
    }

    public static void print(){
        System.out.println("x=" +x +" y=" + y);
    }

    public static ObjInitial2 getInstance(){
        return instance;
    }
}
```

![图片](https://uploader.shimo.im/f/jp4BTbUvtR8MA6rr.png!thumbnail)

### 2.4.8、static初始化使用new关键字
```
public class ClassInitialDemo {
    public static void main(String[] args) throws ClassNotFoundException {
     
       ObjInitial.getInstance().print();
    }
}


/**
 * 注意
 * 初始化顺序如下：
 * 1、instance = null 静态变量赋默认值null
 * 2、x = 0 静态变量赋默认值
 * 3、y = 0 静态变量赋默认值
 * 4、instance = new ObjInitial3() 静态变量，初始化导致构造函数初始化 执行x++ y++
 * 5、x = 0 静态变量初始化
 * 6、static静态代码块初始化， 执行x++ y++
 */
class ObjInitial3{

    private static ObjInitial3 instance = new ObjInitial3();

    private static int x = 0;
    private static int y;


    static {
        x++;
        y++;
        System.out.println("ObjInitial3 static 初始化");
    }

    public ObjInitial3(){
        x++;
        y++;
        System.out.println("ObjInitial3 new 初始化");
    }

    public static void print(){
        System.out.println("x=" +x +" y=" + y);
    }

    public static ObjInitial3 getInstance(){
        return instance;
    }
}
```

![图片](https://uploader.shimo.im/f/7fKRR0rYfFUCV23K.png!thumbnail)

### 2.4.9、静态代码块变量访问规则
```
class ObjInitial4{
    
    static int x = 0;
    
    static {
        // 静态代码块中可以对代码块之前声明的变量进行读写操作 
        System.out.println(x);
        
        x = x + 1;
        
        // 对代码块之后生声明的变量只能进行写操作，不能进行读操作
        y = 1;

         // 这行代码编译器会报错
        // System.out.println(y);
    }
    
    static int y;
    
}
```


## 2.5、jvava内置的类加载器
* **根加载器（****Bootstrap**）， 由c语言实现， 该加载器**没有父级加载器**，他负责加载jvm的核心类库，如java.lang.*,** java.lang.Object就是由根加载器加载的**，根加载器从系统属性sun.boot.class.path所指定的目录中加载类库, 根加载器的**实现依赖于底层操作系统**，**属于java虚拟机实现的一部分，他没有继承java.lang.ClassLoader**
* **扩展类加载器（****ExtClassLoader****)**, 他的**父加载器为根加载器**，他从**java.ext.dirs**系统属性所指定的目录中加载类库或者从jdk的安装目录的jre\lib\ext子目录下加载类库，**如果把用户创建的jar文件放在该目录下，也会自动由扩展加载器加载**。扩展类加载器是纯java类，他继承java.lang.ClassLoader
* **系统（****AppClassLoader****）类加载器**：也称为应用类加载器，它的**父加载器为扩展类加载器**，它从环境变量classpath或者系统属性** java.class.path**所指定的目录中加载类，**他是用户自定义的加载器的默认父加载器**，系统类加载器是纯java类，他继承java.lang.ClassLoader

![图片](https://uploader.shimo.im/f/4RE1spBznHMOu78j.png!thumbnail)

![图片](https://uploader.shimo.im/f/ngLycfztZoMpvg5F.png!thumbnail)

**示例代码：**

```
package classLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 13:57
 */
public class BootClassLoader {

    public static void main(String[] args) {
        //根加载器加载的目录
        System.out.println(System.getProperty("sun.boot.class.path"));

        //扩展类加载器(Extension)加载的目录
        System.out.println(System.getProperty("java.ext.dirs"));

        // 系统（System）类加载器加载的目录
        System.out.println(System.getProperty("java.class.path"));
        
       Class classInitialClass = Class.forName("classLoader.ClassInitialDemo");
      // 打印：sun.misc.Launcher$AppClassLoader@18b4aac2
      // 其加载器是系统加载器AppClassLoader
      System.out.println(classInitialClass.getClassLoader());

     // 打印：sun.misc.Launcher$ExtClassLoader@677327b6
    // 父加载器是ExtClassLoader， 证明了AppClassLoader的父加载器是ExtClassLoader
   System.out.println(classInitialClass.getClassLoader().getParent());

    // 打印：null,
    // ExtClassLoader的父加载器由于是根加载器，不是java语言实现的，
 System.out.println(classInitialClass.getClassLoader().getParent().getParent());

    // 打印:null
    // 说明String是由根加载加载的
    Class stringClass = Class.forName("java.lang.String");
   System.out.println(stringClass.getClassLoader());

    // 打印null
   // 说明Object是由根加载加载的
   Class objClass = Class.forName("java.lang.Object");
   System.out.println(objClass.getClassLoader());
   
    // 打印：sun.misc.Launcher$AppClassLoader@18b4aac2
   // 获取系统类加载器
   ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
   System.out.println(systemClassLoader);
   
    //数组类型的class对象是由jvm创建的，不是由AppClassLoader创建的
    int[] a = new int[1];
    System.out.println(a.getClass());
    System.out.println(a.getClass().getClassLoader());

    // 对于任意类型的数组来说，Class.getClassLoader()的返回值总是和数组中元素的ClassLoader一致
    Obj[] objs = new Obj[1];
    System.out.println(objs.getClass());
    System.out.println(objs.getClass().getClassLoader());


    }
}
```


**注意：对于任意类型的数组来说，数组类型的Class对象不是由ClassLoader创建的，而是由jvm在需要的时候自动创建的，数组类型的对象Class.getClassLoader()方法的返回值总是和数组中元素的ClassLoader一致。**

**ClassLoader源代码文档：**

>Class objects for array classes are not created by class
>loaders, but are created automatically as required by the Java runtime.
>The class loader for an array class, as returned by {@link
>Class#getClassLoader()} is the same as the class loader for its element
>type; if the element type is a primitive type, then the array class has no
>class loader.
>

idea运行结果如下：

```
# 根加载器Bootstrap结果
C:\Program Files\Java\jdk1.8.0_201\jre\lib\resources.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\rt.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\sunrsasign.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jsse.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jce.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\charsets.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jfr.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\classes

# 扩展加载器(ExtClassLoader)结果
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext;
C:\WINDOWS\Sun\Java\lib\ext

# 系统加载器(AppClassLoader)结果
C:\Program Files\Java\jdk1.8.0_201\jre\lib\charsets.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\deploy.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\access-bridge-64.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\cldrdata.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\dnsns.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\jaccess.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\jfxrt.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\localedata.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\nashorn.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunec.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunjce_provider.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunmscapi.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\sunpkcs11.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\ext\zipfs.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\javaws.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jce.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jfr.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jfxswt.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\jsse.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\management-agent.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\plugin.jar;
C:\Program Files\Java\jdk1.8.0_201\jre\lib\resources.jar;
C:\ProgramFiles\Java\jdk1.8.0_201\jre\lib\rt.jar;
F:\code\thread-learn\demo1\target\classes;
F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\junit\junit\4.12\junit-4.12.jar;
F:\program\nexus-2.11.4-01\sonatype-work\nexus\storage\central\org\hamcrest\hamcrest-core\1.3\hamcrest-core-1.3.jar;
C:\Program Files\JetBrains\IntelliJ IDEA 2018.3.4\lib\idea_rt.jar


sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@677327b6
null
null
null
sun.misc.Launcher$AppClassLoader@18b4aac2
class [I
null
class [LclassLoader.Obj;
sun.misc.Launcher$AppClassLoader@18b4aac2
```


## 2.6、ClassLoader加载类的原理
### 2.6.1、 原理   
ClassLoader使用的是**双亲委派机制**来搜索加载类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个组合的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。**当一个ClassLoader实例需要加载某个类时**，**它在亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的**，**首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常**。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

**类加载器双亲委托模型：** 

![图片](https://uploader.shimo.im/f/PTeE4soyk8cYwoOc.png!thumbnail)


### 2.6.2、双亲委托模型好处
因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要 ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。

### 2.6.3、类与类加载器的笔记
类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类加载阶段。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达更通俗一些：比较两个类是否”相等”，只有再这两个类是有同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class 文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

![图片](https://uploader.shimo.im/f/Id827sBOA14qii4f.png!thumbnail)

![图片](https://uploader.shimo.im/f/sCfG5uiKHZIbTaHQ.png!thumbnail)

![图片](https://uploader.shimo.im/f/zrRln0WecHEc11g7.png!thumbnail)

### **2.6.4、JDK的ClassLoader类源代码**
```
// 默认构造方法，可以看出如果没有指定父加载器则默认父加载器为系统加载器AppClassLoader
protected ClassLoader() {
    this(checkCreateClassLoader(), getSystemClassLoader());
}


protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 首先检查这个类是否已经被加载过，如果已经被加载过则直接返回class
        Class<?> c = findLoadedClass(name);
        
        // 还没有被加载过
        if (c == null) { 
            long t0 = System.nanoTime();
            try {
                // 如果有父加载器，则先委托给父加载器去加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } 
                // 没有父加载器，则由根加载器查找
                else {
                    c = findBootstrapClassOrNull(name);
                }
            } 
            catch (ClassNotFoundException e) {
              
            }

            // 没有父加载器或父加载器返回null没有加载成功
            if (c == null) {
             
                long t1 = System.nanoTime();
                // 由自己加载这个类
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        // 返回null或加载的class
        return c;
    }
}
```

先检查是否已经被加载过，若没有加载则调用父加载器的loadClass() 方法，**若父加载器为空，则默认使用启动类加载器作为父加载器**。如果父加载器失败，再调用自己的findClass 方法进行加载,因此到这里再次证明了类加载器的过程： 
## 2.7、自定义ClassLoader实验
### 第1步、编写MyClassLoader.java
```
package classLoader;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

/**
 * 自定义类加载器
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 15:30
 */
public class MyClassLoader extends ClassLoader {

    private String dir;

    /**
     * 如果不指定父ClassLoader则使用系统默认的ExtClassLoader
     * @param dir
     */
    public MyClassLoader(String dir){
        this.dir = dir;
    }

    public MyClassLoader(String dir, ClassLoader parent){
        super(parent);

        this.dir = dir;
    }


    public void setDir(String dir) {
        this.dir = dir;
    }

    public String getDir() {
        return dir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String path = name.replace('.', '/');
        File file = new File(dir, path + ".class");
        if (!file.exists()){
            throw new ClassNotFoundException();
        }

        byte[] classBytes = loadClassBytes(file);
        if (classBytes == null || classBytes.length == 0){
            throw new ClassNotFoundException();
        }

        return this.defineClass(name, classBytes, 0, classBytes.length);
    }

    private byte[] loadClassBytes(File file) {
        try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
             FileInputStream fis = new FileInputStream(file);){

            byte[] buffer = new byte[1024];
            int length = -1;
            while ((length = fis.read(buffer)) != -1){
                bos.write(buffer, 0 , length);
            }

            return bos.toByteArray();
        }
        catch (IOException e) {
            return null;
        }
    }
}
```


### 第2步、编写测试类
```
package classLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 16:10
 */
public class MyClassLoaderDemo {

    public static void main(String[] args) throws ClassNotFoundException {
        String classBaseDir = "C:\\TestTargetClassLoader";
        MyClassLoader myClassLoader = new MyClassLoader(classBaseDir);

        Class clazz = myClassLoader.loadClass("classLoader.MyClassObj
");
        System.out.println(clazz);
        System.out.println(clazz.getClassLoader());
        System.out.println(clazz.getClassLoader().getParent());
        


    }
}
```

编写待加载的类MyClassObj.java

```
package classLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 16:51
 */
public class MyClassObj {
    
    private static int x = 1;
    
    static {
        System.out.println("MyClassObj 执行了静态代码块 x=" +x);
    }
}
```

### 第3步、编译、拷贝及运行
编译MyClassObj.java ，拷贝编译后的target\classed包到磁盘C:/TestTargetClassLoader下，注意保证包结构完整

![图片](https://uploader.shimo.im/f/ZKSifclFTfEOYEgQ.png!thumbnail)

删除当前项目的target目录下的MyClassObj.class文件，因为如果不删除这个文件会导致由于双亲委派模型的加载规则，导致自定义加载器的父加载器先加载到这个文件，那么我们自定义的加载器就不会再次去加载了

![图片](https://uploader.shimo.im/f/Z9MTE1b3gMQ9vcPg.png!thumbnail)

注意：如果运行后仍发现是由AppClassLoader加载的，则去target/classes目录下看下MyClassObj.class是否已经删除，通常是由于运行的时候idea又把MyClassObj.class编译出来了导致的

**最终运行效果如下：**

![图片](https://uploader.shimo.im/f/zbYq4Jl0IFwZcrqB.png!thumbnail)

延伸：类加载器多次加载class的是否得到的是同样的class?

当在MyClassLoaderDemo.java中最后如下代码后:

```
 // 同一个类加载的实例多次调用
MyClassLoader myClassLoader2 = new MyClassLoader(classBaseDir);
Class clazz2 = myClassLoader2.loadClass("classLoader.MyClassObj");
System.out.println("第1个ClassLoader实例加载MyClassObj.class得到的hashcode:" + clazz2.hashCode());

Class clazz3 = myClassLoader2.loadClass("classLoader.MyClassObj");
System.out.println("第1个ClassLoader实例加载MyClassObj.class得到的hashcode:" + clazz3.hashCode());

// 又一个classLoader实例
MyClassLoader myClassLoader3 = new MyClassLoader(classBaseDir);
Class clazz4 = myClassLoader3.loadClass("classLoader.MyClassObj");
System.out.println("第2个ClassLoader实例加载MyClassObj.class得到的hashcode:" + clazz4.hashCode());

```
![图片](https://uploader.shimo.im/f/Ie1YvpBZxmsiK6lY.png!thumbnail)

从结果可以看到：

1、同一个类加载器的**同一个实例多次调用**返回的**Class是完全一样的(hascode相同)**

2、而同一个类加载器的多个**不同实例**加载同一个MyClassObj.class文件得到的Class是不一样的(hashcode不相同)

## 2.8、类的卸载
![图片](https://uploader.shimo.im/f/3LN9H2JRpG4oPZhf.png!thumbnail)

```
MyClassLoader myClassLoader6 = new MyClassLoader(classBaseDir);
Class clazz7 = myClassLoader6.loadClass("classLoader.customClassLoader.MyClassObj");
System.out.println(clazz7.hashCode());
System.out.println(clazz7.getClassLoader());
// 必须
myClassLoader6 = null;
// 必须
```
## clazz7 = null;


System.gc();


设置jvm启动参数   -XX:+TraceClassUnloading 跟踪类的卸载![图片](https://uploader.shimo.im/f/kyp27UPtswAGWIr7.png!thumbnail)

输出如下：

![图片](https://uploader.shimo.im/f/r4ifJuGH8YgZPzQ8.png!thumbnail)

## 2.9、打破双亲委派模型，自下而上加载
### 1、打破双亲委派模型的原理
加载class的顺序是自下而上加载（与jdk的ClassLoader实现不同, jdk默认是自上而下加载的），先由自定义加载器加载，如果自己加载不到则再委托给父加载器加载。

### 2、打破双亲委派模型demo
```
package classLoader.customClassLoader;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 19:47
 */
public class MyClassLoader2 extends ClassLoader {


    private String dir;

    /**
     * 如果不指定父ClassLoader则使用系统默认的ExtClassLoader
     * @param dir
     */
    public MyClassLoader2(String dir){
        this.dir = dir;
    }

    public MyClassLoader2(String dir, ClassLoader parent){
        super(parent);

        this.dir = dir;
    }


    public void setDir(String dir) {
        this.dir = dir;
    }

    public String getDir() {
        return dir;
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // 检查是否已经加载过了
            Class<?> cls = findLoadedClass(name);
            // 还没有加载过
            if (cls == null){
               // 先从自己尝试加载class
                cls = findClass(name);

                // 自己加载不了class，并且有父加载器
                if (cls == null && this.getParent() != null){
                    // 由父加载器加载class
                    cls = this.getParent().loadClass(name);
                }
            }

            return cls;
        }
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String path = name.replace('.', '/');
        File file = new File(dir, path + ".class");
        if (!file.exists()){
            return null;
        }

        byte[] classBytes = loadClassBytes(file);
        if (classBytes == null || classBytes.length == 0){
            throw new ClassNotFoundException();
        }

        return this.defineClass(name, classBytes, 0, classBytes.length);
    }

    private byte[] loadClassBytes(File file) {
        try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
             FileInputStream fis = new FileInputStream(file);){

            byte[] buffer = new byte[1024];
            int length = -1;
            while ((length = fis.read(buffer)) != -1){
                bos.write(buffer, 0 , length);
            }

            return bos.toByteArray();
        }
        catch (IOException e) {
            return null;
        }
    }
}
```


测试：

```
package classLoader.customClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 20:01
 */
public class MyClassLoader2Demo {

    public static void main(String[] args) throws ClassNotFoundException {
        String classBaseDir = "C:\\TestTargetClassLoader";

        MyClassLoader2 myClassLoader21 = new MyClassLoader2(classBaseDir);
        Class clazz21 = myClassLoader21.loadClass("classLoader.customClassLoader.MyClassObj");

        System.out.println(clazz21);
        System.out.println(clazz21.getClassLoader());
        System.out.println(clazz21.getClassLoader().getParent());
    }
}
```

![图片](https://uploader.shimo.im/f/c8JmSQrAAdsBNGN0.png!thumbnail)


### **3、打破双亲委派模型后的自定义加载器是否能够加载java.lang.String?**
创建一个新项目，结构如下：

![图片](https://uploader.shimo.im/f/d3trZgqQUxU5GHJG.png!thumbnail)

把String.class编译后按照其包结构放到之前的C:\TestTargetClassLoader目录下

![图片](https://uploader.shimo.im/f/yQRgK5txfG4wVzyo.png!thumbnail)

上述MyClassLoader2Demo .java文件最后加入如下代表测试是否能够加载String类

```
Class stringClass = myClassLoader21.loadClass("java.lang.String");
System.out.println(stringClass);
```

![图片](https://uploader.shimo.im/f/wlGg13KvsO8zI3ZG.png!thumbnail)

报错了，说明并不能加载到String.class文件

## 2.10 获得ClassLoader的途径
![图片](https://uploader.shimo.im/f/dYyHWYx5XlwEpOwf.png!thumbnail)

## 2.11、类加载的命名空间
### 2.11.1、概念
![图片](https://uploader.shimo.im/f/hwoZTyP0n8gcWGK6.png!thumbnail)

**结论：**

1、子加载器(MyClassLoader)所加载的类(MySimple)能访问父加载器(AppClassLoader)加载的类(MyCat)

2、父加载器(AppClassLoader)所加载的类(MyCat)不能访问子加载器(MyClassLoader)所加载的类(MySimple)

3、一个类的类加载是由其所在调用方法的类加载器加载的


### 2.11.2、示例1
```
package classLoader.namespace;

import classLoader.customClassLoader.MyClassLoader2;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/10 20:41
 */
public class NamespaceDemo {

    public static void main(String[] args) throws IllegalAccessException, InstantiationException, ClassNotFoundException {
        String classBaseDir = "C:\\TestTargetClassLoader";

        MyClassLoader2 myClassLoader21 = new MyClassLoader2(classBaseDir);
        Class clazz21 = myClassLoader21.loadClass("classLoader.namespace.NamespaceObj");

        System.out.println(clazz21.hashCode());
        System.out.println(clazz21.getClassLoader());
        
        System.out.println();
        
        System.out.println(NamespaceObj.class.hashCode());
        System.out.println(NamespaceObj.class.getClassLoader());
        
        NamespaceObj myClassObj = (NamespaceObj) clazz21.newInstance();
        System.out.println(myClassObj);

    }
}
```


输出：

![图片](https://uploader.shimo.im/f/QgOnTxyoa3k6oqdS.png!thumbnail)

从输出可以看到ClassLoader不一样， NamespaceObj myClassObj = (NamespaceObj) clazz21.newInstance();这行赋值会出现ClassCastException, 明明都是NamespaceObj, 但是其hashcode不同，实例化时当然不是同一个类

### 2.11.3、一个类的类加载是由其所在调用方法的类加载器加载的
1、代码编写

```
package classLoader.namespace;

import classLoader.customClassLoader.MyClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 20:14
 */
public class NamespaceDemo2 {

    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        String classBaseDir = "C:\\TestTargetClassLoader";

        MyClassLoader myClassLoader = new MyClassLoader(classBaseDir);
        Class mySimpleClass = myClassLoader.loadClass("classLoader.namespace.MySimple");
        System.out.println(mySimpleClass.getClassLoader());

        Object mySimple = mySimpleClass.newInstance();
    }
}
```



MySimple.java文件

```
package classLoader.namespace;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 21:00
 */
public class MySimple{
    public MySimple(){
        System.out.println("MySimple invoked");

        System.out.println("MySimple ClassLoader:" + this.getClass().getClassLoader());
        new MyCat();
    }
}
```


MyCat.java文件

```
package classLoader.namespace;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 21:00
 */
public class MyCat {

    public MyCat(){
        System.out.println("MyCat invoked");

        System.out.println("MyCat ClassLoader:" + this.getClass().getClassLoader());
    }
}
```


2、编译完后复制class文件到非Classpath目录，如下：

![图片](https://uploader.shimo.im/f/bwUfNhT9NxUexuNi.png!thumbnail)

3、然后删除target/classLoader/namespace中的MyCat.class文件

![图片](https://uploader.shimo.im/f/OTXlAB0MFesvCfbe.png!thumbnail)

4、运行结果

![图片](https://uploader.shimo.im/f/UU27OkPaeC8ndtjO.png!thumbnail)

**说明：**

可以看到明明自己是用MyClassLoader.loadClass("classLoader.namespace.MySimple")去加载MySimple.class文件，由于target目录下有MySimple.class文件，所以根据双亲委派机制实际的加载**MySimple.class文件****是由AppClassLoader加载的**，这个毫无疑问，大家都懂。

**但是明明****C:\TestTargetClassLoader\classLoader\namespace里面有MyCat.class文件，而target没有MyCat.class文件却报错ClassNotFoundException,为什么呢？**

原因：MySimple类的构造方法中 new MyCat();这一行代码去实例化会导致加载MyCat.class文件，那么MyCat.class文件是由哪个ClassLoader加载的呢? **是由AppClassLoader加载的**，**因为MyCat的实例化是在MySimple类中进行的，所以会使用和当前调用方（MySimple）相同的类加载器(AppClassLoader)去加载MyCat.class**, 而target目录下并没有MyCat.class文件所以导致抛出了**ClassNotFoundException文件。**

所以得出开始的结论：一个类的类加载是由其所在调用方法的类加载器加载的

### 2.11.4、命名空间示例1：继续改造
重新编译上述后, 删除target/classLoader/namespace中的MySimple.class文件,保留MyCat.class文件

![图片](https://uploader.shimo.im/f/WE6FZae587ciziid.png!thumbnail)

重新运行结果如下：

![图片](https://uploader.shimo.im/f/I0zh7dj1BnESAhWm.png!thumbnail)

说明：MySimple.class是由MyClassLoader加载的， 在MySimple.java的构造方法中new Mycat()这一行，首先由MyClassLoader加载，由于双亲委派机制会委托给AppClassLoader加载，AppClassLoader委托给ExtClassLoader, ExtClassLoader委托给Bootstrap ClassLoader, 而根加载器和Ext加载器都加载不到，所以会回到AppClassLoader加载，而target目录下有Mycat.class文件，所以成功加载MyCat类

### 2.11.5、**父加载器****(AppClassLoader)****所加载的类****(MyCat)****不能访问子加载器****(MyClassLoader)****加载的类****(MySimple)**
1、代码

```
package classLoader.namespace;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 21:00
 */
public class MySimple{

    public MySimple(){
        System.out.println("MySimple invoked");

        System.out.println("MySimple ClassLoader:" + this.getClass().getClassLoader());

        new MyCat();

        //子加载器(MyClassLoader)所加载的类(MySimple)能访问父加载器(AppClassLoader)加载的类(MyCat)
//        System.out.println("from MySimple: " + MyCat.class);
    }
}
```


```
package classLoader.namespace;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 21:00
 */
public class MyCat {

    public MyCat(){
        System.out.println("MyCat invoked");

        System.out.println("MyCat ClassLoader:" + this.getClass().getClassLoader());

        //父加载器(AppClassLoader)所加载的类(MyCat)不能访问子加载器(MyClassLoader)加载的类(MySimple)
        System.out.println("from MyCat" + MySimple.class);
    }
}
```



2、重新编译，把target/classLoader/namespace下的文件和之前一样复制到非classpath目录

![图片](https://uploader.shimo.im/f/nnz5YMuRK2wixjaS.png!thumbnail)

3、删除MySimple.class, 保留MyCat.class文件，重新运行

![图片](https://uploader.shimo.im/f/cOa7CFUjgPIRKPPu.png!thumbnail)

说明是由于刚刚在MyCat.java的构造方法中加入的System.*out*.println("from mycat" + MySimple.class);这一行代码导致的。

原因：MySimple.class由MyClassLoader加载， MyCat.class由AppClassLoader加载, 在MyCat.class中使用MySimple.class会导致加载MySimple.class, 虽然之前加载过MySimple.class， 但是之前的MySimple.class是在MyClassLoader中加载的，而现在MySimple.class是由AppClassLoader尝试加载，而target目录下的MySimple.class已经被我们删除了，所以抛出ClssNotFoundException

### 2.1.6、子加载器(MyClassLoader)所加载的类(MySimple)能访问父加载器(AppClassLoader)加载的类(MyCat)
1、代码

```
public class MySimple{

    public MySimple(){
        System.out.println("MySimple invoked");

        System.out.println("MySimple ClassLoader:" + this.getClass().getClassLoader());

        new MyCat();

        //子加载器(MyClassLoader)所加载的类(MySimple)能访问父加载器(AppClassLoader)加载的类(MyCat)
        System.out.println(MyCat.class);
    }
}
```
```
package classLoader.namespace;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/11 21:00
 */
public class MyCat {

    public MyCat(){
        System.out.println("MyCat invoked");

        System.out.println("MyCat ClassLoader:" + this.getClass().getClassLoader());

        //父加载器(AppClassLoader)所加载的类(MyCat)不能访问子加载器(MyClassLoader)加载的类(MySimple)
//        System.out.println("from MyCat" + MySimple.class);
    }
}
```


2、重新编译、复制拷贝到之前的非classpath目录，

![图片](https://uploader.shimo.im/f/nnz5YMuRK2wixjaS.png!thumbnail)

3、然后删除target/classLoader/namespace目录下的MySimple.class，保留MyCat.class，重新运行

![图片](https://uploader.shimo.im/f/dEkqlhQyk3kdsfct.png!thumbnail)

## 2.12、替换系统类加载器
### 2.12.1、通过java.system.class.loader属性替换
ClssLoader.getSysemClassLoader()方法的javadoc文档如下：

![图片](https://uploader.shimo.im/f/2iOFZ8KPrLcknPZT.png!thumbnail)

1、编写如下代码测试

```
package classLoader.customClassLoader;

/**
 * 替换系统类加载器
 * 
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/12 22:09
 */
public class ReplaceSytemClassLoader {

    public static void main(String[] args) {
             System.out.println(System.getProperty("java.system.class.loader"));
        System.out.println(MyClassObj.class.getClassLoader());
        System.out.println(ClassLoader.getSystemClassLoader());

    }
}
```


输出如下：可以看到java.system.class.loader的值是null， 此时系统类加载器是AppClassLoader

![图片](https://uploader.shimo.im/f/Rg6EkmkAtQMy0JPQ.png!thumbnail)

2、进入到target.classes目录下执行:

 java **-Djava.system.class.loader**=classLoader.customClassLoader.MyClassLoader classLoader.customClassLoader.ReplaceSytemClassLoader

输出如下：可以看到java.system.class.loader确实已经被改变了, 而ClassLoader.*getSystemClassLoader*()的结果已经变成我们设置的MyClassLoader类加载器了

![图片](https://uploader.shimo.im/f/Sf7K1AB7RYsgr540.png!thumbnail)

### **2.12.2、注意：要被替换为系统类加载器的自定义类加载必须定义无参数的构造方法**
![图片](https://uploader.shimo.im/f/iN273wk0x2MeMvc8.png!thumbnail)

假如把这个构造方法注释掉会报错， 这个构造方法是由La

![图片](https://uploader.shimo.im/f/EP9bIEeusMAskzPN.png!thumbnail)

## 2.13、ContextClassLoader
### 2.13.1、入门示例
```
package classLoader.contextClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/13 20:55
 */
public class ContextClassLoaderDemo {

    public static void main(String[] args) {
        System.out.println(Thread.class.getClassLoader());
        System.out.println(Thread.currentThread().getContextClassLoader());
    }
}
```

![图片](https://uploader.shimo.im/f/mFHwgFBibSAbaNhm.png!thumbnail)

原因：Thread类是由启动类加载器加载的，所以打印null, 当前类是位于classpath下的，由AppClassLoader加载的

### 2.13.2、当前类加载器
概念：加载当前类的类加载器

**每个类都会使用自己的类加载器（即加载自身的类加载器）来去加载其所依赖的类，如果ClassX引用了ClassY,那么ClassX的类加载器就会去加载ClassY(前提是ClassY尚未被加载)**

代码示例：

CurrentClassLoader .java

```
package classLoader.contextClassLoader;

import classLoader.customClassLoader.MyClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/13 21:03
 */
public class CurrentClassLoader {

    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        String classBaseDir = "C:\\TestTargetClassLoader";
        MyClassLoader myClassLoader = new MyClassLoader(classBaseDir);

        Class clazz = myClassLoader.loadClass("classLoader.contextClassLoader.ClassA");
        clazz.newInstance();
    }
}
```


ClassA.java

```
package classLoader.contextClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/13 21:04
 */
public class ClassA {

    public ClassA(){
        System.out.println("ClassA: " + this.getClass().getClassLoader());

        new ClassB();
    }
}
```


ClassB.java

```
package classLoader.contextClassLoader;

/**
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/13 21:04
 */
public class ClassB {

    public ClassB(){
        System.out.println("ClassB: " + this.getClass().getClassLoader());
    }
}
```

然后把target/classLoader/contextClassLoader下面的类复制到非classpath目录

![图片](https://uploader.shimo.im/f/PAOioL7YUBou7wYT.png!thumbnail)

再删除当前项目的target/classLoader/contextClassLoader下面的ClassA.class、ClassB.class

最后运行输出如下：

![图片](https://uploader.shimo.im/f/nd66iUyfJpIwtHAq.png!thumbnail)

可以看到ClassA是由MyClassLoader加载的（这个毫无疑问，再不懂就说不过去了），而在ClassA的构造方法中实例化了ClassB, 从输出看到ClassB也是由MyClassLoader加载的，验证前面的结论：**每个类都会使用自己的类加载器（即加载自身的类加载器）来去加载其所依赖的类**

### 2.13.3、线程上下文类加载器
线程上下文类加载器是从jdk1.2开始引入的，类Thread中的getContextClassLoader()与setContextClassLoader(ClassLoader cl)分别用来获取和设置上下文类加载器。
如果没有通过setContextClassLoader(ClassLoader cl)进行设置的话，线程将继承父线程的上下文类加载器，java线程运行时的上下文类加载器是系统类加载器，在线程中运行的代码可以通过该类加载器来加载类与资源。
    

线程上下文类加载器的重要性：
    
SPI（Service Provider Interface）
    
父ClassLoader可以使用当前线程Thread.currentThread().getContextClassLoader()所指定的ClassLoader加载的类，这就改变了父ClassLoader不能使用子ClassLoader或是其他没有直接父子关系的classLoader加载的类的情况，即改变了双亲委托模型。

线程上下文类加载器就是当前线程的current ClassLoader。
    
双亲委托模型是自下而上的，即下层的类加载器会委托给上层的类加载器进行加载，但是对于SPI来说，有些接口是由java核心库提供的，而java核心库是由启动类加载器加载的，而这些接口的实现却来自于不同的jar包（厂商提供），java的启动类加载器是不会加载其他来源的jar包，这样传统的双亲委托模型就无法满足SPI的要求，而通过给当前线程设置上下文类加载器（将当前线程上下文的类加载设置为接口的类加载器），那么接口的实现提供方(厂商提供的jar包)就可以获取到当前线程的上下文类加载器，来用和接口一样的类加载器去加载接口的实现，解决接口的类加载器和实现类加载器不一致的问题。

### 2.1.4、线程上下文类加载器的一般使用模式
```
package classLoader.contextClassLoader;

import classLoader.customClassLoader.MyClassLoader;

/**
 * context classloader的一般使用模式
 * 
 * @author calebzhao<9 3 9 3 4 7 5 0 7 @ qq.com>
 * 2019/8/13 22:04
 */
public class ContextClassLoaderUseDemo {

    public static void main(String[] args) {
        String classBaseDir = "C:\\TestTargetClassLoader";
        MyClassLoader myClassLoader = new MyClassLoader(classBaseDir);
        invoke(myClassLoader);
    }

    /**
     * context classloader使用模式
     * @param classLoader
     */
    public static void invoke(ClassLoader classLoader){
        ClassLoader originClassLoader = Thread.currentThread().getContextClassLoader();

        try{
            Thread.currentThread().setContextClassLoader(classLoader);
            // doBusiness()
        }
        finally {
            Thread.currentThread().setContextClassLoader(originClassLoader);
        }
    }
}
```


