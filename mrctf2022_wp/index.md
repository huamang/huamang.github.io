# [MRCTF2022] WP


## Misc

### checkin

翻源码找到flag

![image-20220423093224882](https://tuchuang.huamang.xyz/img/image-20220423093224882.png)

##  Web

### WebCheckIn

上传东西上去发现没有回显，然后试着传其他的文件上去，发现随手写的php文件上传上去了，返回了路径

然后就往里面加了一句话木马进去了

![image-20220423134056156](https://tuchuang.huamang.xyz/img/image-20220423134056156.png)

然后发现根目录没有flag

![image-20220423134126222](https://tuchuang.huamang.xyz/img/image-20220423134126222.png)

而且只是www权限，猜测是不是要提权，看了一下php文件的权限就是root

![image-20220423134226603](https://tuchuang.huamang.xyz/img/image-20220423134226603.png)

于是直接用php文件反弹shell，直接反弹shell不行，会报错，这里用的curl反弹shell

![image-20220423134319048](https://tuchuang.huamang.xyz/img/image-20220423134319048.png)

进去直接find搜flag，成功拿到

![image-20220423134401997](https://tuchuang.huamang.xyz/img/image-20220423134401997.png)

## Bonus

### Java_mem_shell_Basic

打开是tomcat的默认页面，进入manage看看，弱口令tomcat:tomcat登录成功，进了manage就可以随便getshell了

这里把冰蝎马打包一个war包，然后传上去拿到shell

```
jar cvf hmnb.war shellmh.jsp
```

然后连进冰蝎，而且给的直接就是root权限

![image-20220424074214197](https://tuchuang.huamang.xyz/img/image-20220424074214197.png)

感觉冰蝎的终端不好用，反弹一个shell

![image-20220424074508620](https://tuchuang.huamang.xyz/img/image-20220424074508620.png)

直接find查找到flag

![image-20220424075140362](https://tuchuang.huamang.xyz/img/image-20220424075140362.png)

### Java_mem_shell_Filter

进入是一个登录页面

![image-20220424075502027](https://tuchuang.huamang.xyz/img/image-20220424075502027.png)

目录扫描没啥结果

![image-20220424080136132](https://tuchuang.huamang.xyz/img/image-20220424080136132.png)

尝试爆破一个账号密码，爆破了一大堆，找不到可用的

尝试sql注入，试了很多，根本试不出报错，尝试跑一下sqlmap，还是无果

后来发现是log4j。。。

![image-20220424092433941](https://tuchuang.huamang.xyz/img/image-20220424092433941.png)

马上来打一波

![image-20220424095851667](https://tuchuang.huamang.xyz/img/image-20220424095851667.png)

进去以后，发现web目录有一个godzilla，密码拿到了，密钥是默认的key

![image-20220424125343511](https://tuchuang.huamang.xyz/img/image-20220424125343511.png)

连进哥斯拉里面

![image-20220424125415146](https://tuchuang.huamang.xyz/img/image-20220424125415146.png)

find没找到flag，结合题目名理解，可能是在filter内存马里面，这里用jmapdump出jvm内存

首先查看pid： `ps -ef|grep java`

![image-20220424180756412](https://tuchuang.huamang.xyz/img/image-20220424180756412.png)

然后dump：`jmap -dump:format=b,file=e.bin 51`

然后下载出来

![image-20220424175248636](https://tuchuang.huamang.xyz/img/image-20220424175248636.png)

找到flag

![image-20220425190844505](https://tuchuang.huamang.xyz/img/image-20220425190844505.png)

