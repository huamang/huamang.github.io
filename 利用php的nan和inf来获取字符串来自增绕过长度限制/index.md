# 利用PHP的NAN和INF来获取字符串来自增绕过长度限制




最近在[一道题目](https://blog.csdn.net/m0_51078229/article/details/119745997)里面学到了一个新的小技巧
在php中，字符串是可以递增的，如下：
但是不可递减

![在这里插入图片描述](https://img-blog.csdnimg.cn/b8e678eff65441339beeb3465388113b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzUxMDc4MjI5,size_16,color_FFFFFF,t_70)

这道题是这样的

```php
<?php
error_reporting(0);
if ($_GET['looklook']){
    highlight_file(__FILE__);
}else{
    setcookie("hint", "?looklook", time()+3600);
}
if (isset($_POST['ctf_show'])) {
    $ctfshow = $_POST['ctf_show'];
    if (is_string($ctfshow) || strlen($ctfshow) <= 107) {
        if (!preg_match("/[!@#%^&*:'\"|`a-zA-BD-Z~\\\\]|[4-9]/",$ctfshow)){
            eval($ctfshow);
        }else{
            echo("fucccc hacker!!");
        }
    }
} else {

    phpinfo();
}
?>
```
可以看到，这里长度的限制是107，我们如果拼接一个_GET，如果硬从C开始自增，那可能会超过限制，T理C太远了，所以这里学到了两个新的操作

在[这篇文章](https://blog.csdn.net/rfrder/article/details/119535886)有写到（也是原题）
```
$_=C/C ---> NAN
$_=1/C ---> INF
```
但是这不是string类型，但在php里面，我们可以通过拼接一个字符串来让他转成String类型
比如这里我写如下代码

```php
<?php

$a = (C/C.C);
$b = (1/C.C);
var_dump($a);
var_dump($b);
```


![在这里插入图片描述](https://img-blog.csdnimg.cn/e6200a7ca3c24f22b06d941f6df7deca.png)

成功得到string类型，这里我们就使用索引就可以单选中字母了

![在这里插入图片描述](https://img-blog.csdnimg.cn/d06cdfac3dec4eb5aa85650fe9aacfa0.png)

这样做就可以从N开始递增了，避开了长度的限制

