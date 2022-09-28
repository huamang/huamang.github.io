# [Java安全] Java反序列化


## 前言

在Java安全中，Java反序列化占据很重要的地位。

在前面RMI的探索中，我们可以在数据包中看到，他传递的数据是序列化的

![image-20220928172800137](https://tuchuang.huamang.xyz/img/image-20220928172800137.png)

Java的反序列化和PHP的反序列化其实有点类似，他们都只能将一个对象中的属性按照某种特定的格式 生成一段数据流，在反序列化的时候再按照这个格式将属性拿回来，再赋值给新的对象

但Java相对PHP序列化更深入的地方在于，其提供了更加高级、灵活地方法 writeObject ，**允许开发者在序列化流中插入一些自定义数据**，进而在反序列化的时候能够使用 readObject 进行读取



## 反序列化基础

要比较完美的实现反序列化的操作，有两个注意点

1. 如果想让一个对象能够序列化，那么就要实现一个特殊的接口，这个接口就是`java.io.Serializable`，这个接口是空的，相当于是一个**标记**作用，意味着实现了这个接口的类可以进行序列化/反序列化
2. 设置`serialVersionUID`，在进行反序列化时，JVM 会把传来的字节流中的 serialVersionUID 与本地相应实体类的 serialVersionUID 进行比较，如果相同就认为是一致的，可以进行反序列化，否则就抛出序列化版本不一致的异常- InvalidCastException



再来说说`writeObject()`和`readObject()`

在writeObject中，得先创建一个`ObjectOutputStream`，然后利用这个把对象写入字节流，然后调用writeObject来序列化对象，代码实现如下

- 创建对象
- 创建文件流
- 创建对象输入流
- 把对象序列化写入对象输入流
- 释放资源关闭流

```java
public static void serializing(String filename) throws IOException {
    serializeObj obj = new serializeObj("Huamang");
    FileOutputStream file = new FileOutputStream(filename);
    ObjectOutputStream out = new ObjectOutputStream(file);
    out.writeObject(obj);
    out.close();
    file.close();
}
```

readObject就用对应的ObjectIntputStream和FileInputStream，读取加载在文件中的序列化对象并对其反序列化

```java
public static void unserializing(String filename) throws IOException, ClassNotFoundException {
    FileInputStream file = new FileInputStream(filename);
    ObjectInputStream in = new ObjectInputStream(file);
    serializeObj testclass = (serializeObj) in.readObject();
    testclass.test();
}
```

然后写主类进行函数调用，最后的代码是这样的

```java
import java.io.*;

public class serializeTest {
    
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        String filename = "./data.txt";
        serializeTest.serializing(filename);
        serializeTest.unserializing(filename);
    }

    public static void serializing(String filename) throws IOException {
        serializeObj obj = new serializeObj("Huamang");
        FileOutputStream file = new FileOutputStream(filename);
        ObjectOutputStream out = new ObjectOutputStream(file);
        out.writeObject(obj);
        out.close();
        file.close();
    }

    public static void unserializing(String filename) throws IOException, ClassNotFoundException {
        FileInputStream file = new FileInputStream(filename);
        ObjectInputStream in = new ObjectInputStream(file);
        serializeObj testclass = (serializeObj) in.readObject();
        testclass.test();
    }
}

```

```java
import java.io.Serializable;

public class serializeObj implements Serializable {
    public String name;
    public serializeObj(String name){
        this.name = name;
    }
    public void test(){
        System.out.println("This is "+name);
    }
}

```

成功反序列化获得对象并执行方法

![image-20220928225655695](https://tuchuang.huamang.xyz/img/image-20220928225655695.png)

## 对序列化流的分析

这里分析一下这个序列化数据，先转hex，然后利用工具

![image-20220928225640940](https://tuchuang.huamang.xyz/img/image-20220928225640940.png)

```
java -jar SerializationDumper-v1.13.jar aced00057372000c73657269616c697a654f626a2ef7c37a55dbe7910200014c00046e616d657400124c6a6176612f6c616e672f537472696e673b78707400074875616d616e67
```

获得结果

```
STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_OBJECT - 0x73
    TC_CLASSDESC - 0x72
      className
        Length - 12 - 0x00 0c
        Value - serializeObj - 0x73657269616c697a654f626a
      serialVersionUID - 0x2e f7 c3 7a 55 db e7 91
      newHandle 0x00 7e 00 00
      classDescFlags - 0x02 - SC_SERIALIZABLE
      fieldCount - 1 - 0x00 01
      Fields
        0:
          Object - L - 0x4c
          fieldName
            Length - 4 - 0x00 04
            Value - name - 0x6e616d65
          className1
            TC_STRING - 0x74
              newHandle 0x00 7e 00 01
              Length - 18 - 0x00 12
              Value - Ljava/lang/String; - 0x4c6a6176612f6c616e672f537472696e673b
      classAnnotations
        TC_ENDBLOCKDATA - 0x78
      superClassDesc
        TC_NULL - 0x70
    newHandle 0x00 7e 00 02
    classdata
      serializeObj
        values
          name
            (object)
              TC_STRING - 0x74
                newHandle 0x00 7e 00 03
                Length - 7 - 0x00 07
                Value - Huamang - 0x4875616d616e67
```

可以看到我的name是绑定了传入的数据"Huamang"的，想到前面说到：允许开发者在序列化流中插入一些自定义数据

首先借鉴了feng师傅的一个图

![image-20220928232255297](https://tuchuang.huamang.xyz/img/image-20220928232255297.png)

我们的obj是实现了Serializable接口的，但是里是空的，但是这里告诉我们，我们是可以重写readObject方法的，这样我们可以把恶意代码写入这里，然后被反序列化的时候就会执行我们构造的恶意代码，比如这样重写

```java
    private void readObject(java.io.ObjectInputStream stream)
            throws IOException, ClassNotFoundException{
        stream.defaultReadObject();
        Runtime.getRuntime().exec("calc");
    }
```

看起来他的作用有点像PHP中的 `__wakeup`，但是他们的设计理念是不一样的

>Java设计 readObject 的思路和PHP的 __wakeup 不同点在于： readObject 倾向于解决“反序列化时如 何还原一个完整对象”这个问题，而PHP的 __wakeup 更倾向于解决“反序列化后如何初始化这个对象”的 问题。

同样的，我们writeObject也是可以重写自定义的

废话不多说我们直接上代码理解

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class serializeTest2 implements Serializable {
    public String name;
    public int id;

    public serializeTest2(String name, int id) {
        this.name = name;
        this.id = id;
    }

    private void writeObject(ObjectOutputStream os) throws IOException {
        os.defaultWriteObject();
        os.writeObject("Write Object");
    }

    private void readObject(ObjectInputStream os) throws IOException, ClassNotFoundException {
        os.defaultReadObject();
        System.out.println(os.readObject());
    }
}

```

然后修改刚刚序列化与反序列化的方法，把序列化的类换成这个类

```java
import java.io.*;

public class serializeTest {
    
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        String filename = "./data.txt";
        serializeTest.serializing(filename);
        serializeTest.unserializing(filename);
    }

    public static void serializing(String filename) throws IOException {
        serializeTest2 obj = new serializeTest2("Huamang",1);
        FileOutputStream file = new FileOutputStream(filename);
        ObjectOutputStream out = new ObjectOutputStream(file);
        out.writeObject(obj);
        out.close();
        file.close();
    }

    public static void unserializing(String filename) throws IOException, ClassNotFoundException {
        FileInputStream file = new FileInputStream(filename);
        ObjectInputStream in = new ObjectInputStream(file);
        serializeTest2 testclass = (serializeTest2) in.readObject();
    }
}

```

执行并分析序列化数据

```
STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_OBJECT - 0x73
    TC_CLASSDESC - 0x72
      className
        Length - 14 - 0x00 0e
        Value - serializeTest2 - 0x73657269616c697a655465737432
      serialVersionUID - 0xfd 59 ea 3d 16 b4 a0 7b
      newHandle 0x00 7e 00 00
      classDescFlags - 0x03 - SC_WRITE_METHOD | SC_SERIALIZABLE
      fieldCount - 2 - 0x00 02
      Fields
        0:
          Int - I - 0x49
          fieldName
            Length - 2 - 0x00 02
            Value - id - 0x6964
        1:
          Object - L - 0x4c
          fieldName
            Length - 4 - 0x00 04
            Value - name - 0x6e616d65
          className1
            TC_STRING - 0x74
              newHandle 0x00 7e 00 01
              Length - 18 - 0x00 12
              Value - Ljava/lang/String; - 0x4c6a6176612f6c616e672f537472696e673b
      classAnnotations
        TC_ENDBLOCKDATA - 0x78
      superClassDesc
        TC_NULL - 0x70
    newHandle 0x00 7e 00 02
    classdata
      serializeTest2
        values
          id
            (int)1 - 0x00 00 00 01
          name
            (object)
              TC_STRING - 0x74
                newHandle 0x00 7e 00 03
                Length - 7 - 0x00 07
                Value - Huamang - 0x4875616d616e67
        objectAnnotation
          TC_STRING - 0x74
            newHandle 0x00 7e 00 04
            Length - 12 - 0x00 0c
            Value - Write Object - 0x5772697465204f626a656374
          TC_ENDBLOCKDATA - 0x78
```

可以看到我们`writeObject`的参数”Write Object“被放在了`objectAnnotation`下

而在我反序列化的时候，执行`readObject`，读取到了这个数据并输出

![image-20220928234022766](https://tuchuang.huamang.xyz/img/image-20220928234022766.png)

