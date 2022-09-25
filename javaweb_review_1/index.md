# Javaweb_Review_1


# Javaweb基础复习

# JDBC基础操作

要点：

- 导入mysql的jar包，在顶栏中选择项目，模块，导入jar
- `Class.forName("com.mysql.jdbc.Driver");` 初始化给定的类，自动注册驱动
- 获取连接con
- 利用con获取sql语句对象statement，编写sql语句，执行sql语句
- 遍历结果集getString获取结果内容
- 释放资源

代码实现

```java
package com.jdbctest;

import org.junit.Test;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class JDBCDemo1 {
    // 使用junit的test来调试
    @Test

    public void demo1() throws Exception{
        // 加载驱动
        Class.forName("com.mysql.jdbc.Driver");

        // 获得连接,把结果放到Connection类型的对象里面，后面的两个参数是数据库的用户名和密码
        Connection con = DriverManager.getConnection("jdbc:mysql://localhost:3306/java","root","");
        // 基本操作
            // 获取sql语句的对象
        Statement statement = con.createStatement();
            // 编写sql语句
        String sql = "select * from test";
            // 执行sql语句,把结果放到ResultSet类型的对象里面
        ResultSet res = statement.executeQuery(sql);
            // 遍历结果集,打印出id，username，passwd
        while (res.next()){
            System.out.print(res.getString("id")+' ');
            System.out.print(res.getString("username")+' ');
            System.out.print(res.getString("passwd"));
            System.out.println();
        }
        // 释放资源，依次释放
        res.close();
        statement.close();
        con.close();
    }
}
```

# JDBC工具类提取

因为注册资源，操作完再释放资源，这些重复的操作非常傻逼，所以这里这里提取出utils

对于驱动的注册集合在一个方法里

对于连接的获取放在一个方法里

对于资源的释放放在一个方法里

因为数据库操作会有很多的变数，所以必须手动操作

```java
import org.junit.Test;

import java.sql.*;

public class JDBC_Utils {
    // 变量声明
    private static final String driverClassName;
    private static final String url;
    private static final String username;
    private static final String password;
    // 配置定义
    static{
        driverClassName="com.mysql.jdbc.Driver";
        url="jdbc:mysql:///java";
        username="root";
        password="123456";
    }

    //驱动注册
    public static void loadDriver() throws ClassNotFoundException {
        Class.forName("com.mysql.jdbc.Driver");
    }

    // 获得连接
    public static Connection getConn() throws SQLException, ClassNotFoundException {
        loadDriver();
        Connection con = DriverManager.getConnection(url,username,password);
        return con;
    }

    // 释放资源
    public static void release(ResultSet res, Statement statement, Connection con) throws SQLException {
        res.close();
        statement.close();
        con.close();
    }

    @Test
    public void Test() throws SQLException, ClassNotFoundException {
        Connection con = getConn();
        String sql = "select * from user";
        Statement statement = con.createStatement();
        ResultSet res = statement.executeQuery(sql);
        while(res.next()){
            System.out.printf(res.getString("id")+' ');
            System.out.printf(res.getString("name")+' ');
            System.out.printf(res.getString("passwd")+' ');
        }
        release(res,statement,con);
    }

}
```

# 连接池 Druid

![image-20220926022140873](https://tuchuang.huamang.xyz/img/image-20220926022140873.png)

首先为什么要有连接池这么个东西？

在生产环境里，对于数据库的访问的非常频繁的，可能访问一个页面就调用了好几次数据库了

那么如果按以前的方法，每次进行数据库操作，都去获取连接，释放连接，这会非常消耗资源，尤其是在人多的情况下，甚至出现宕机

所以这里映入数据库链接库，使用就申请，用完就放回

数据库连接池的实现是有厂商做好了的，而且实现的很好，这里就使用Druid来学习使用

- 导入jar包 druid-1.0.9.jar
- 定义配置文件：
    - 是properties形式的可以叫任意名称
    - 可以放在任意目录下
- 加载配置文件:Properties
- 获取数据库连接池对象：通过工厂类来获取 DruidDataSourceFactory
- 获取连接：getConnection 利用重载
- 释放：重写close 分两个情况，一个是有res的，一个是没有的

properties文件示例，踩坑，路径问题，推荐放在src目录下

```java
driverClassName=com.mysql.jdbc.Driver
url=jdbc:mysql:///java
username=root
password=123456
initialSize=5
maxActive=10
maxWait=3000
```

代码实现工具类提取

```java
import com.alibaba.druid.pool.DruidDataSourceFactory;
import org.junit.Test;

import javax.sql.DataSource;
import java.io.IOException;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.Properties;

public class JDBC_Druid {
    private static DataSource ds;

    static {
        //加载配置文件
        Properties properties = new Properties();
        try {
            // 参数为文件路径，建议放在src目录下
            properties.load(JDBC_Druid.class.getClassLoader().getResourceAsStream("druid.properties"));
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 获取连接池对象
        try {
            ds = DruidDataSourceFactory.createDataSource(properties);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 获取连接
    public static Connection getConnection() throws SQLException {
        return ds.getConnection();
    }

    // 释放资源：执行sql语句没有res的情况
    public static void close(Statement statement,Connection con) throws SQLException {
        statement.close();
        con.close();
    }

    // 释放资源：执行sql语句没有res的情况
    public static void close(ResultSet res,Statement statement,Connection con) throws SQLException {
        res.close();
        statement.close();
        con.close();
    }

    @Test
    public void Test() throws SQLException {
        String sql = "select * from user";
        Connection con = JDBC_Druid.getConnection();
        Statement statement = con.createStatement();
        ResultSet res = statement.executeQuery(sql);
        while(res.next()){
            System.out.println(res.getString("id"));
            System.out.println(res.getString("name"));
            System.out.println(res.getString("passwd"));
        }
        JDBC_Druid.close(res,statement,con);
    }

}
```

# JDCB最终章****Template****

Spring对JDBC做了简单封装，提供了JDBCTemplate对象简化JDBC的操作，JDBC的操作变得更加简单方便了！

## 1. 导包

首先还得是导包，JdbcTemplate位于spring-jdbc中。其全限定命名为org.springframework.jdbc.core.JdbcTemplate。要使用JdbcTemlate还需一个spring-tx这个包包含了一下事务和异常控制，还有必要的spring和log组件

![image-20220926022153761](https://tuchuang.huamang.xyz/img/image-20220926022153761.png)

## 2. 创建JdbcTemplate对象

下一步就是创建JdbcTemplate对象，依赖于数据源DataSource（数据库连接池）

Template依赖于数据源DataSource，所以这里得创建DataSource，为了方便，我这里直接提取Druid的工具类，创建了个函数去直接获取DataSource

JDBC_Druid.java中定义如下方法

```java
// 获取连接池，由于在static代码块中已经实例化好了连接池对象，所以可以直接返回
public static DataSource getDataSource(){
    return ds;
}
```

```java
JdbcTemplate template = new JdbcTemplate(JDBC_Druid.getDataSource());
```

## 3. 调用方法完成CRUD操作

1. 调用JdbcTemplate的方法来完成CRUD的操作

```jsx
update():执行DML语句。增、删、改语句，第二个参数是预处理的填充

queryForMap():查询结果将结果集封装为map集合，将列名作为key，将值作为value 将这条记录封装为一个map集合
注意：这个方法查询的结果集长度只能是1

queryForList():查询结果将结果集封装为list集合
注意：将每一条记录封装为一个Map集合，再将Map集合装载到List集合中

query():查询结果，将结果封装为JavaBean对象（通常使用这个方法）
	query的参数：RowMapper
		一般我们使用BeanPropertyRowMapper实现类。可以完成数据到JavaBean的自动封装
		new BeanPropertyRowMapper<类型>(类型.class)

queryForObject：查询结果，将结果封装为对象
一般用于聚合函数的查询
```

## 实操

这里以queryForMap为例，这里将结果封装为map集合

```java
import org.junit.Test;
import org.springframework.jdbc.core.JdbcTemplate;
import java.util.Map;

public class Template {
    /**
     * Template依赖于数据源DataSource，所以这里得创建DataSource，为了方便，我这里直接提取Druid的工具类，创建了个函数去直接获取DataSource
     */
    @Test
    public void test() {
        JdbcTemplate template = new JdbcTemplate(JDBC_Druid.getDataSource());
        String sql = "select * from user";
        Map<String, Object> map = template.queryForMap(sql);
        System.out.println(map);
    }
}
```

至于之类为啥是 `Map<String, Object>` 他的value为啥是Object，我跟进了一下源码，这里mapRow就是这样写的返回值

![image-20220926022204064](https://tuchuang.huamang.xyz/img/image-20220926022204064.png)

当然你不确定的时候可以选择不写先，等写完后面的再利用idea编辑器的功能来补全

![image-20220926022211639](https://tuchuang.huamang.xyz/img/image-20220926022211639.png)
