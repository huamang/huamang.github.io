# Fastjson




# Fastjson

fastjson 是阿里巴巴的开源 JSON 解析库，它可以解析 JSON 格式的字符串，**支持将 Java Bean 序列化为 JSON 字符串，也可以从 JSON 字符串反序列化到 JavaBean**。

## 序列化与反序列化

首先了解他的序列化与反序列化，他的使用十分的简单

```java
//序列化 JavaBean -> JSON
String text = JSON.toJSONString(obj); 
//反序列化 JSON -> JavaBean
VO vo = JSON.parse(); //解析为JSONObject类型或者JSONArray类型
VO vo = JSON.parseObject("{...}"); //JSON文本解析成JSONObject类型
VO vo = JSON.parseObject("{...}", VO.class); //JSON文本解析成VO.class类
```

做个简单演示


