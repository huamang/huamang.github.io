# CISCN2022_Semifinals




# ikari

在index.php里面有这么一段代码

```php
try
{
    require ROOT_PATH . 'Engine/Loader.php';
    E\Loader::getInstance()->init(); // Load necessary classes
    $aParams = ['ctrl' => (!empty($_GET['p']) ? $_GET['p'] : 'blog'), 'act' => (!empty($_GET['a']) ? $_GET['a'] : 'index'), 'template' => (!empty($_GET['t']) ? $_GET['t'] : 'pc'), 'ns' => (!empty($_GET['n']) ? $_GET['n'] : 'TestProject\Controller\\')];
    
    E\Router::run($aParams);
}
catch (\Exception $oE)
{
    echo $oE->getMessage();
}


?>
```

我们跟进Router去看看

```php
class Router
{
    public static function run (array $aParams)
    {
        if($aParams['ns'] != 'TestProject\Controller\\'){
            $sNamespace = $aParams['ns'];
            $sCtrlPath = $sNamespace;
        }else{
            $sNamespace = 'TestProject\Controller\\';
            $sDefCtrl = $sNamespace . 'Blog';
            $sCtrlPath = ROOT_PATH . 'Controller/';
        }
        if($aParams['ns'] == '\\'){
            $aParams['ctrl'] = '\\system';
            if(preg_match('/^[a-zA-Z0-9\/]*$/',$aParams['act'])){
                $aParams['act'] = 'ls /www/'.$aParams['act'];
            }else{
                $aParams['act'] = 'ls /www/';
            }
        }
        $sTemplatePath = str_replace(array(".","\/"), "", ROOT_PATH . 'Template/' . $aParams['template']);
        include $sTemplatePath;
        $sCtrl = ucfirst($aParams['ctrl']);
        if (is_file($sCtrlPath . $sCtrl . '.php') || (substr($sCtrl, 0, 1) === '\\'))
        {

            $sCtrl = $sNamespace . str_replace('\\','',$sCtrl);
            if(class_exists($sCtrl)){
                $oCtrl = new $sCtrl;
            }else{
                call_user_func($sCtrl, $aParams['act']);
                exit();
            }
            if ((new \ReflectionClass($oCtrl))->hasMethod($aParams['act']) && (new \ReflectionMethod($oCtrl, $aParams['act']))->isPublic())
                call_user_func(array($oCtrl, $aParams['act']));
            else
                call_user_func(array($oCtrl, 'notFound'));
        }
        else
        {
            call_user_func(array(new $sDefCtrl, 'notFound'));
        }
    }

}
```

这里是对路由的操作，这里，我们可以通过控制参数p选择路由，a选择路由的功能

在utils里面有这个，这里可以进行文件的包含

```php
public function __get($name)
{
  if(stripos($name,'php') || stripos($name,'Upload') || stripos($name,'flag') || stripos($name,':')){
    die('Dangerous Operation.');
  }else{
    if(file_exists($name)){
      require $name;
    }
  }
}
```

而这个函数的入口在blog路由下的post方法，这里传一个参数f，进行包含，因为这里对参数有过滤，所以我们选择session_upload进行竞争包含

```php
public function post()
{
  if($this->render == 0){
    $this->oUtil->oPost = $this->oModel->getById($this->_iId); // Get the data of the post

    $this->oUtil->getView('post');
  }else{
    $this->oUtil->__get($_GET['f']);
  }

}
```



![image-20220624205433693](https://tuchuang.huamang.xyz/img/image-20220624205433693.png)



![image-20220624205443833](https://tuchuang.huamang.xyz/img/image-20220624205443833.png)



![image-20220624205458083](https://tuchuang.huamang.xyz/img/image-20220624205458083.png)

# xxxcloud

源码中有admin的密码hash，但是解不开，这里的做法是找到token的构造算法，进行手动token生成



![image-20220624205655175](https://tuchuang.huamang.xyz/img/image-20220624205655175.png)

找到token计算算法



![image-20220624205712736](https://tuchuang.huamang.xyz/img/image-20220624205712736.png)

找到systempassword



![image-20220624205737985](https://tuchuang.huamang.xyz/img/image-20220624205737985.png)

计算出token



![image-20220624205752225](https://tuchuang.huamang.xyz/img/image-20220624205752225.png)

然后伪造登录获取flag

这里的防守不知道应该用什么方式进行防守，如果是伪造token的话，那我们无法避免，因为token算法是写死的，我尝试把这个系统的一个任意文件删除修复，但是任然是exp攻击成功

后来直接上了个awd的通防waf，没想到还防住了。。

# template

任意文件读取 + 读 opcache缓存

这里传入tpl参数，赋值为$tpl

```php
if (isset($_GET['tpl']) && is_string($_GET['tpl'])):
    $tpl = $_GET['tpl'];
endif;
```

然后在check方法里面，先进行后缀检测，这里过滤的是挺严的

如果通过了过滤，那么就会进行file_get_contents函数，进行文件读取

```php
    private function check() {
        try{
            $arr = explode('.', $this->tpl);
            $ext = end($arr);
            if (in_array($ext, ['php', 'php2', 'php3', 'php4', 'php5', 'php6', 'php7', 'phtml'])):
                return false;
            endif;

            $content = file_get_contents($this->tpl);
            if (!$content):
                return false;
            endif;

            if ( preg_match('/script|<\?/i', $content) ):
                return false;
            endif;
        } catch (Exception $e) {
            return false;
        }

        return true;
    }
```

那我们就可以进行除了php文件外，任意文件的读取了

这里如果tpl是debug的话，会给phpinfo



![image-20220624210641434](https://tuchuang.huamang.xyz/img/image-20220624210641434.png)



这里是开启了zend_opcache，这里我们利用包含opcache进行攻击

首先得计算systemid，这个的计算可以利用工具https://github.com/GoSecure/php7-opcache-override/



![image-20220624210840281](https://tuchuang.huamang.xyz/img/image-20220624210840281.png)



然后就是任意文件读取了

http://172.16.9.44:8091/?tpl=..//...//...//...//.../var/www/cache/1116d566fdc53f79abce6c01e3a0308d/var/www/html/flag.php.bin

