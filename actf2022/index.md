# [ACTF2022] WP




# Web

## gogogo

先看dockerfile

```
FROM debian:buster

RUN set -ex \
    && sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list && sed -i 's/security.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list \
    && apt-get update \
    && apt-get install wget make gcc -y \
    && wget -qO- https://github.com/embedthis/goahead/archive/refs/tags/v5.1.4.tar.gz | tar zx --strip-components 1 -C /usr/src/ \
    && cd /usr/src \
    && make SHOW=1 ME_GOAHEAD_UPLOAD_DIR="'\"/tmp\"'" \
    && make install \
    && cp src/self.key src/self.crt /etc/goahead/ \
    && mkdir -p /var/www/goahead/cgi-bin/ \
    && apt-get purge -y --auto-remove wget make gcc \
    && cd /var/www/goahead \
    && rm -rf /usr/src/ /var/lib/apt/lists/* \
    && sed -e 's!^# route uri=/cgi-bin dir=cgi-bin handler=cgi$!route uri=/cgi-bin dir=/var/www/goahead handler=cgi!' -i /etc/goahead/route.txt

COPY flag /flag
RUN chmod 644 /flag
COPY hello /var/www/goahead/cgi-bin/hello
RUN chmod +x /var/www/goahead/cgi-bin/hello

RUN groupadd -r ctf && useradd -r -g ctf ctf
EXPOSE 8081

USER ctf
CMD ["goahead", "-v", "--home", "/etc/goahead", "/var/www/goahead", "0.0.0.0:8081"]
```

这里是用了一个goahead，而这里的hello路由放在了cgi-bin里面，而且输出了env

![image-20220626132410787](https://tuchuang.huamang.xyz/img/image-20220626132410787.png)

那么就不难找到是gohead的CVE-2021-42342，这个cve的内容是文件上传过滤器的处理缺陷，当与CGI处理程序一起使用时，可影响环境变量，从而导致RCE

记得p牛复现这个漏洞的时候踩坑专门写了篇[文章](https://www.leavesongs.com/PENETRATION/goahead-en-injection-cve-2021-42342.html) 

这和题目dockerfile如出一辙

![image-20220626133113981](https://tuchuang.huamang.xyz/img/image-20220626133113981.png)

而且我们的确可以传文件

![image-20220626133209612](https://tuchuang.huamang.xyz/img/image-20220626133209612.png)

接下来就是劫持LD_PRELOAD，对于这个东西的了解可以参考如下

>LD_PRELOAD 是 Linux 系统中的一个环境变量，它可以影响程序的运行时的链接（Runtime linker），它允许你定义在程序运行前优先加载的动态链接库。这个功能主要就是用来有选择性的载入不同动态链接库中的相同函数。通过这个环境变量，我们可以在主程序和其动态链接库的中间加载别的动态链接库，甚至覆盖正常的函数库。一方面，我们可以以此功能来使用自己的或是更好的函数（无需别人的源码），而另一方面，我们也可以以向别人的程序注入程序，从而达到特定的目的。

我们编译一个so文件，因为可以操控env，那么我们传上去绑定LD_PRELOAD，然后就可以达到攻击的目的，但是我始终没复现成功

利用p牛复现的时候的技巧，添加脏字符让长度超过content-length还是没有起作用，不知道是不是没有注意到什么细节

![image-20220626133719771](https://tuchuang.huamang.xyz/img/image-20220626133719771.png)

后来发现p牛其实还发过一个[文章](https://www.leavesongs.com/PENETRATION/how-I-hack-bash-through-environment-injection.html)，也是关于环境变量攻击的



![image-20220626134124422](https://tuchuang.huamang.xyz/img/image-20220626134124422.png)

这里用`BASH_FUNC_xxx%%=(){id;}`，把原函数给覆盖掉

然后看到hello里面，他是执行了env的，所以我们把env给覆盖掉

```
BASH_FUNC_env%%:(None,"() { cat /flag; exit; }")
```

即注册一个这样变量，让他执行env的时候，变成执行我们定义的函数内容



![image-20220626134505028](https://tuchuang.huamang.xyz/img/image-20220626134505028.png)

