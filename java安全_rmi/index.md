# [Java安全]RMI


# 前言

RMI全程`Remote Method Invocation`，远程⽅法调⽤

他的功能是让一个Java虚拟机上的对象调用另一个Java虚拟机中对象上的方法，是Java独有的一种机制，他是由三层架构模式来实现的

- Client-客户端：客户端调用服务端的方法
- Server-服务端：远程调用方法对象的提供者，也是代码真正执行的地方，执行结束会返回给客户端一个方法执行的结果。
- Registry-注册中心：其实本质就是一个map，相当于是字典一样，用于客户端查询要调用的方法的引用

>为了屏蔽网络通信的复杂性，RMI 引入了两个概念，分别是 Stubs（客户端存根） 以及 Skeletons（服务端骨架），当客户端（Client）试图调用一个在远端的 Object 时，实际调用的是客户端本地的一个代理类（Proxy），这个代理类就称为 Stub，而在调用远端（Server）的目标类之前，也会经过一个对应的远端代理类，就是 Skeleton，它从 Stub 中接收远程方法调用并传递给真实的目标类。Stubs 以及 Skeletons 的调用对于 RMI 服务的使用者来讲是隐藏的，我们无需主动的去调用相关的方法。但实际的客户端和服务端的网络通信时通过 Stub 和 Skeleton 来实现的。

![image-20220927162458903](https://tuchuang.huamang.xyz/img/image-20220927162458903.png)

![image-20220927211311444](https://tuchuang.huamang.xyz/img/image-20220927211311444.png)

RMI Registry就像⼀个**⽹关**，他⾃⼰是不会执⾏远程⽅法的，但RMI Server可以在上⾯注册⼀个Name 到对象的绑定关系；RMI Client通过Name向RMI Registry查询，得到这个绑定关系，然后再连接RMI Server；最后，远程⽅法实际上在RMI Server上调⽤。



# RMI简单实现

## Server

首先从Server来下手：

⼀个RMIServer分为三部分： 

1. ⼀个继承了 java.rmi.Remote 的接⼝，其中定义我们要远程调⽤的函数，⽐如这⾥的 hello() 
2. ⼀个实现了此接⼝的类 
3. ⼀个主类，⽤来创建Registry，并将上⾯的类实例化后绑定到⼀个地址。这就是我们所谓的Server 了。



首先是接口实现并继承Remote，里面创建一个hello方法，抛出RemoteException异常

```java
public interface IRemoteHelloWorld extends Remote {
 public String hello() throws RemoteException;
 }
```

然后创建一个类去实现这个接口**用于远程调用**

继承UnicastRemoteObject类，貌似继承了之后会使用默认socket进行通讯，并且该实现类会一直运行在服务器上。如果不继承UnicastRemoteObject类，则需要手工初始化远程对象，在远程对象的构造方法的调用UnicastRemoteObject.exportObject()静态方法。

```java
package com.example.RMI;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class RMIServer extends UnicastRemoteObject implements IRemoteHelloWorld {

    protected RMIServer() throws RemoteException {
        super();
    }

    @Override
    public String hello() throws RemoteException {
        return "hello world!";
    }

}
```

现在可以被远程调用的对象被创建好了，接下来就是思考该如何调用他了

> Java RMI 设计了一个 Registry 的思想，很好理解，我们可以使用注册表来查找一个远端对象的引用，更通俗的来讲，这个就是一个 RMI 电话本，我们想在某个人那里获取信息时（Remote Method Invocation），我们在电话本上（Registry）通过这个人的名称 （Name）来找到这个人的电话号码（Reference），并通过这个号码找到这个人（Remote Object）。

这种电话本的思想，由 `java.rmi.registry.Registry` 和 `java.rmi.Naming` 来实现。这里分别来说说这两个东西。

先来说说 `java.rmi.Naming`，这是一个 final 类，提供了在远程对象注册表（Registry）中存储和获取远程对象引用的方法，这个类提供的每个方法都有一个 URL 格式的参数，格式如下：` //host:port/name`：

- host 表示注册表所在的主机
- port 表示注册表接受调用的端口号，默认为 1099
- name 表示一个注册 Remote Object 的引用的名称，不能是注册表中的一些关键字



那么这样就好理解了，我们现在实现了服务端待调用的对象，现在我们要把他装载进“电话本”，也就是注册

这里是这样的步骤

1. 利用`LocateRegistry.createRegistry(1099);`创建注册中心
2. 实例化远程对象
3. 把这个实例化对象绑定name存入“电话本”

```java
package com.example.RMI;

import java.net.MalformedURLException;
import java.rmi.AlreadyBoundException;
import java.rmi.Naming;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;

public class RemoteServer {
    public static void main(String[] args) throws RemoteException, MalformedURLException, AlreadyBoundException {
        //创建注册中心
        LocateRegistry.createRegistry(1099);
        //创建远程对象
        RMIObject rmiObject = new RMIObject();
        // 绑定name
        Naming.bind("rmi://localhost:1099/Hello", rmiObject);

    }
}
```

这样就算是把服务端给简单搭建好了

## Client

现在来搭建服务器端，进行远程加载服务端的代码，服务端加载远程代码的步骤：

1. 使用`Naming` 在`Registry`中寻找到名字是Hello的对象
2. 调用远程对象的方法属性



Naming有很多方法

![image-20220928002633273](https://tuchuang.huamang.xyz/img/image-20220928002633273.png)

这里用lookup来测试，他返回：a reference for a remote object，远程对象的引用

- 此`Naming.lookup()`调用检查在 localhost 中运行的 RMI 注册表中是否存在名为“Hello”的绑定
- 它返回一个必须转换为我期望的任何远程接口的对象
- 然后就可以使用该对象调用接口中定义的远程方法

![image-20220928003106171](https://tuchuang.huamang.xyz/img/image-20220928003106171.png)

```java
package com.example.RMI;

import java.net.MalformedURLException;
import java.rmi.Naming;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;

public class RMIClient {
    public static void main(String[] args) throws MalformedURLException, NotBoundException, RemoteException {
        IRemoteHelloWorld obj = (IRemoteHelloWorld) Naming.lookup("rmi://127.0.0.1:1099/Hello");
        System.out.println(obj.hello());
    }
}

```

![image-20220928014249327](https://tuchuang.huamang.xyz/img/image-20220928014249327.png)

这样就算简单的完成了一个RMI的操作

远程也可以加载成功（虚拟机）

![image-20220928020228917](https://tuchuang.huamang.xyz/img/image-20220928020228917.png)





# RMI攻击

## 先留坑



------

https://xz.aliyun.com/t/9261

https://xz.aliyun.com/t/6660

https://redmango.top/article/70

https://paper.seebug.org/1091/

https://su18.org/post/rmi-attack

