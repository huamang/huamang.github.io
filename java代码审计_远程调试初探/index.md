# [Java安全] 远程调试初探




# Jar包的调试
这里以冰蝎做示例
把冰蝎的jar包和他必须的db数据库文件放到项目的lib目录下
把jar包添加到库

![在这里插入图片描述](https://img-blog.csdnimg.cn/ca74b2f745cc4232996dffd0b62458e3.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_18,color_FFFFFF,t_70,g_se,x_16)

如下配置调试配置
新建远程JVM调试
JVM的命令行参数，根据JDK的版本去选择

![在这里插入图片描述](https://img-blog.csdnimg.cn/d0dc2d1df8544067a4796b6c01dd1a1c.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

然后比如在这打一个断点

![在这里插入图片描述](https://img-blog.csdnimg.cn/2da3f6ccea4b46ccb030c7d153d6056b.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

然后在jar包目录下执行如下命令，后面的参数是刚刚配置调试的那个配置
<font color=red>注意这里，suspend要改成y，不然会卡不住断点，这个参数的意思是：是否等待调试器的接入</font>

![在这里插入图片描述](https://img-blog.csdnimg.cn/03c27e4f599e4769b39b38637506c2c8.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

回到idea，debug就可以卡住断点了

![在这里插入图片描述](https://img-blog.csdnimg.cn/2a7bdadcc06f4ee69ed3bb44fe0713fd.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

# Weblogic的远程调试
这里我们是以dacker来配合Idea进行代码调试，使用的是vulhub项目
示例：CVE-2017-10271
在docker-compose.yml里面加上8453端口，这个端口是用于调试的

![在这里插入图片描述](https://img-blog.csdnimg.cn/b2ba81df7eaf4e47adee14a6361fad83.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

`docker-compose up -d`获取镜像并启动后，进入命令行，修改配置文件
在如下路径的setDomainEnv.sh文件中，找到JAVA_DEBUG这里，在下面追加如下代码

```
enbugFlag="true"
export debugFlag
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/faa43d9e5b1f4c1ab43fba4af02c3f12.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

然后执行docker restart xxx命令，重启容器

下一步我们需要把源代码提取到本地
我们需要/root/Oracle/Middleware/下的modules和wlserver_10.3文件

使用cp命令的缺点：操作一些长文件名的时候会报错
推荐使用zip压缩后，cp提取出压缩包

![在这里插入图片描述](https://img-blog.csdnimg.cn/77d4c898563c41b6ae50d8d9a1f3a121.png)

打开idea，将wlserver_10.3/server/lib添加为库

![在这里插入图片描述](https://img-blog.csdnimg.cn/65038a37b7114ee29aa63c414ce30795.png)

配置调试器，和上面的jar包一样，端口换成8453就行了

# Tomcat的远程调试
以CVE-2017-12615漏洞环境示例
开启5005调试端口，我这里把8080换成了1002，防止端口冲突，本地还是开了挺多服务的

![在这里插入图片描述](https://img-blog.csdnimg.cn/daace6fc2e914393a034f7bd16f07a90.png)

访问1002端口，成功部署

![在这里插入图片描述](https://img-blog.csdnimg.cn/82d739f079784aeb8da19e7575e83a11.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

在catalina.sh里面插入如下代码
/usr/local/tomcat/bin/catalina.sh

```php
JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
```
然后重启docker
**注意⚠️ ，这里不能直接拿vscode改**，重启会报错找不到catalina.sh文件。。。踩坑找了好久才找到原因
用echo写进去就行，也不知道是为啥，希望有了解过的师傅解答一下

下一步就是导出docker里面的lib
```
docker cp 79:/usr/local/tomcat/lib ./
```
在idea，创建项目，把lib添加为库

![在这里插入图片描述](https://img-blog.csdnimg.cn/6dbae08c06f340818fcd830df5ed355c.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_11,color_FFFFFF,t_70,g_se,x_16)

配置调试器还是一样的，端口换成5005

![在这里插入图片描述](https://img-blog.csdnimg.cn/57b03c01b5e044b9bf541380dc4b0b62.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

成功卡住

![在这里插入图片描述](https://img-blog.csdnimg.cn/bcf12c67d03e49dda9cb05684aec3560.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAaHVhbWFuZ2dn,size_20,color_FFFFFF,t_70,g_se,x_16)

