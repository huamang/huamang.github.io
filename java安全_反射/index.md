# [Java安全]反射


# 前言
准备入手学习java的安全了，感觉这也是一个大的趋势，想着尽早进入到java安全的探索中，在反序列化链的学习之前，需要先学习反射，不多说了，开干吧

# 反射
## 反射定义
对象可以通过反射获取他的类，类可以通过反射拿到所有⽅法（包括私有）
通过java语言中的反射机制可以操作字节码文件，可以读和修改字节码文件
## 反射的基本运用
### 1. 获取类对象
#### a. forName()方法

只需要知道类名，在加载JDBC的时候会采用
实例代码
```java
public class test1 {
    public static void main(String[] args) throws ClassNotFoundException {
        Class name = Class.forName("java.lang.Runtime");
        System.out.println(name);
    }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/2566ff1a93cf44d8977983c04fbd013b.png)

#### b. 直接获取
使用`.class`去获取对于的对象
```java
public class test1 {
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> name = Runtime.class;
        System.out.println(name);
    }
}
```

#### c. getClass()方法
getClass来获取字节码对象，必须要明确具体的类，然后创建对象
```java
public class test1 {
    public static void main(String[] args) throws ClassNotFoundException {
        Runtime rt = Runtime.getRuntime();
        Class<?> name = rt.getClass();
        System.out.println(name);
    }
}

```
#### d. getSystemClassLoader().loadClass()方法
这个方法和forName类似，只要有类名就可以了，但是区别在于，forName的静态JVM会装载类，并执行static()中的代码

```java
public class getSystemClassLoader {
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> name = ClassLoader.getSystemClassLoader().loadClass("java.lang.Runtime");
        System.out.println(name);
    }
}
```
### 2. 获取类方法
#### a. getDeclaredMethods
返回类或接口声明的所有方法，包括public、protected、private和默认方法，但是不包括继承的方法

```java
import java.lang.reflect.Method;

public class getDeclaredMethods {
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> name = Class.forName("java.lang.Runtime");
        System.out.println(name);
        Method[] m = name.getDeclaredMethods();
        for(Method x:m)
            System.out.println(x);
    }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/8ec532ec19184237be6dc890b081a196.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

#### b. getDeclaredMethod
获取特定的方法，第一个参数是方法名，第二个参数是该方法的参数对应的class对象，例如这里Runtime的exec方法参数为一个String，所以这里的第二个参数是String.class

```java
import java.lang.reflect.Method;

public class getDeclaredMethod {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException {
        Class<?> name = Class.forName("java.lang.Runtime");
        Method m = name.getDeclaredMethod("exec",String.class);
        System.out.println(m);
    }
}
```

#### c. getMethods
返回某个类所有的public方法，包括继承类的public方法
#### d. getMethod
参数同理getDeclaredMethod


### 3. 获取成员变量
同理Method的那几个方法
#### a. getDeclaredFields
获取类的成员的所有变量数组，但是不包括父类的
#### b. getDeclaredField(String name)
获取特定的，参数是想要的方法的名称
#### c. getFields()
同理，只能获得public的，但是包括了父类的
#### d. getField(String name)
同理，参数是想要的方法的名称


### 4. 获取构造函数Constructor

```java
Constructor<?>[] getConstructors() ：只返回public构造函数

Constructor<?>[] getDeclaredConstructors() ：返回所有构造函数

Constructor<> getConstructor(类<?>... parameterTypes) : 匹配和参数配型相符的public构造函数

Constructor<> getDeclaredConstructor(类<?>... parameterTypes) ： 匹配和参数配型相符的构造函数
```
后面两个方法的参数是对于方法的参数的类型的class对象，和Method的那个类似，例如`String.class`





### 5. 反射创建类对象
#### newInstance
可以通过反射来生成实例化对象，一般我们使用Class对象的`newInstance()`方法来进行创建类对象

创建的方法就是：只需要通过forname方法获取到的class对象中进行newInstance方法创建即可

```java
Class c = Class.forName("com.reflect.MethodTest"); // 创建Class对象
Object m1 =  c.newInstance(); // 创建类对象
```
#### invoke
invoke方法位于java.lang.reflect.Method类中，用于<font color=red>执行某个的对象的目标方法</font>,一般会和getMethod方法配合进行调用。

使用用法：

```java
public Object invoke(Object obj, Object... args)
```
第一个参数为类的实例，第二个参数为相应函数中的参数

obj：从中调用底层方法的对象，必须是实例化对象
args： 用于方法的调用，是一个object的数组，参数有可能是多个

但需要注意的是，invoke方法第一个参数并不是固定的：

- **如果调用这个方法是普通方法，第一个参数就是类对象；**

- **如果调用这个方法是静态方法，第一个参数就是类；**

通过一个例子去理解

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Invoke {
    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        Class c = Class.forName("Invoke");
        Object o = c.newInstance();
        Method m = c.getMethod("test");
        m.invoke(o);
    }
    public void test(){
        System.out.println("测试成功");
    }
}

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/1ebd5c7d63104bccbf48b0ab98708375.png)

简单来说就是这样
```java
方法.invoke(类或类对象)
```
先forName拿到Class，再newInstance获取类对象，再getMethod获取方法，然后调用

## Runtime的rce例子（访问限制突破）
Runtime类里面有一个exec方法，可以执行命令

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Exec {
    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        Class c = Class.forName("java.lang.Runtime");
        Object o = c.newInstance();
        Method m = c.getMethod("exec",String.class);
        m.invoke(o,"/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }
}
```
但是发现报错了

![在这里插入代码片](https://img-blog.csdnimg.cn/1303337d47a04902a8a971cdc2009a1c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

出现这个问题的原因：
1. 使用的类没有无参构造函数
2. 使用的类构造函数是私有的


那么解决方案就是`setAccessible(true);`，用这个去突破访问限制
>Java.lang.reflect.AccessibleObject类是Field，Method和Constructor类对象的基类，可以提供将反射对象标记为使用它抑制摸人Java访问控制检查的功能，同时上述的反射类中的Field，Method和Constructor继承自AccessibleObject。所以我们在这些类方法基础上调用setAccessible()方法，既可对这些私有字段进行操作

简单来说，私有的属性、方法、构造方法，可以通过这个去突破限制，`xxx.setAccessible(true)`
可以看到Runtime的构造方法是private的

![在这里插入图片描述](https://img-blog.csdnimg.cn/6fb8f828c4f44083b1fc2190af6040bd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_13,color_FFFFFF,t_70,g_se,x_16)

那么这里我们就可以这么去突破限制
先获取构造方法，然后setAccessible获取访问权限
然后再最后invoke里面，第一个参数写成con.newInstance()
```java
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Exec {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException, InstantiationException {
        Class c = Class.forName("java.lang.Runtime");
        Constructor con = c.getDeclaredConstructor();
        con.setAccessible(true);
        Method m = c.getMethod("exec",String.class);
        m.invoke(con.newInstance(),"/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }
}

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/10431445cf734dacaeb98a41b0f0e77d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里有一个疑问，如果把con.newInstance单独提取出来，他打开计算器不会显示出来，但是后台的确是启动了，不知道啥原因

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Exec {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException, InstantiationException {
        Class c = Class.forName("java.lang.Runtime");
        Constructor con = c.getDeclaredConstructor();
        con.setAccessible(true);
        Object o = con.newInstance();
        Method m = c.getMethod("exec",String.class);
        m.invoke(o,"/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }
}
```

## 后记
反射中常用的几个重要方法：
- 获取类的⽅法： forName
- 实例化类对象的⽅法： newInstance
- 获取函数的⽅法： getMethod
- 执⾏函数的⽅法： invoke
- 限制突破方法：setAccessible




