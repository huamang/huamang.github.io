# [代码审计]ThinkPHP5的文件包含漏洞




# 漏洞影响范围
加载模版解析变量时存在变量覆盖问题，导致文件包含漏洞的产生
漏洞影响版本：**5.0.0<=ThinkPHP5<=5.0.18 、5.1.0<=ThinkPHP<=5.1.10**

# tp框架搭建
tp框架由两部分组成
应用项目: https://github.com/top-think/think
核心框架: https://github.com/top-think/framework
框架下载好后，需要更名为thinkphp

这里可以直接用composer来获取代码
通过以下命令获取测试环境代码：

```
composer create-project --prefer-dist topthink/think=5.0.18 tp5.0.18
```
将 composer.json 文件的 require 字段设置成如下：
```json
"require": {
    "php": ">=5.6.0",
    "topthink/framework": "5.0.18"
},
```
然后执行 composer update ，并将 application/index/controller/Index.php 文件代码设置如下：
```php
<?php
namespace app\index\controller;
use think\Controller;
class Index extends Controller
{
    public function index()
    {
        $this->assign(request()->get());
        return $this->fetch(); // 当前模块/默认视图目录/当前控制器（小写）/当前操作（小写）.html
    }
}
```
创建 application/index/view/index/index.html 文件

# 漏洞分析
payload：

```
http://127.0.0.1/tp5.0.18/public/index?cacheFile=php://filter/read=convert.base64-encode/resource=/etc/passwd
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/7b31190fa31b4a1e87075144d220fc6f.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
打断点进行调试
先进入assign进行模板变量的赋值

![在这里插入图片描述](https://img-blog.csdnimg.cn/b172a46b5515432c9d3b7ef3ba3bec52.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/f9ec8769cb0549fb89e30513f121689e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这边会有一个过滤，看起来像sql的过滤，与我们这次的没啥关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/a823bf0bc5f6470280317c52fb561143.png)

然后进入fetch方法，走完前面的缓存，就会进入一个read方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/951a39d6ee7a4f7b884bbb8c2ab65a64.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_17,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/f3602d22c4bb44dc9e9222c44b7a5637.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里进行了一个变量的赋值：`extract($vars, EXTR_OVERWRITE);`，但是这里他设置了一个属性：`EXTR_OVERWRITE`
这个属性的存在，会照成一个变量覆盖的效果

![在这里插入图片描述](https://img-blog.csdnimg.cn/da67cd8e225d4fd18212b00913c295e4.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

这时候，cacheFile就会变成我们get提交的数据
下一步就是直接包含了这个文件
输出读取的文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/42b63db8b6e1473599c702b91efc9b51.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

成功的包含到了

![在这里插入图片描述](https://img-blog.csdnimg.cn/cb6c51add313431799df1b903f14e0f8.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)



# 漏洞总结
总体来说，漏洞的思路就是：
1. 先通过view的assign把get数据存在数组data中

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/3dad672feb5344d5811e5dd52e1908ca.png)

2. 然后进入view的fetch方法，通过`$vars = array_merge(self::$var, $this->data, $vars);`，将data中的数据再合并到vars的变量中去

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/c793f086b79948fdb247c935d8cb194a.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)


3. 再进入了Think的fetch方法，vars变量被赋进去了

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/9cd157055c2e47d3a7507ad5715d1b4a.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

4. 再到template的fetch方法中去，data变量依旧继续传递下去

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/af65ea17b74f4a7983bc81f10603302f.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

5. 到template的fetch方法里面，里面有一个cacheFile变量，值为箭头所指的文件，到了read方法，data和cacheFile一起被赋值进去

  

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/64822dd09c894df4a548412ab1e1593b.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

6. 可以看到，这里的extract函数由于EXTR_OVERWRITE设置了参数，所以cacheFile变量被覆盖了，导致后面的include就包含了我们vars中的值，而这个值我们是可以控制的，这就造成了一个文件包含的漏洞

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/ef2d178b69e34abe84cc4493c11336d1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
# 漏洞的修复
官方漏洞的修复方案
他这里是用了一个`$this->cacheFile`
这样一来，在include的时候就不会因为变量覆盖而包含到我们data中的数据了
```php
public function read($cacheFile, $vars = [])
{
    $this->cacheFile = $cacheFile;
    if (!empty($vars) && is_array($vars)) {
        // 模板阵列变量分解成为独立变量
        extract($vars, EXTR_OVERWRITE);
    }
    //载入模版缓存文件
    include $this->cacheFile;
}
```
这里[学长的文章](https://www.ieven762.cn/index.php/archives/101/)也是提到了为啥参数不换成`EXTR_SKIP`防止变量的覆盖呢，EXTR_SKIP的作用就是：**如果有冲突，不覆盖已有的变量**，那么这一步就会失效，万一你确实是想传这么一个\$cacheFile=xxx，那么到这里就会直接实效掉，导致功能的损坏，所以利用`$this->cacheFile`是更优的选择

