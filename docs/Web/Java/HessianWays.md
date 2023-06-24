# Hessian 反序列化

## 流程分析

序列化
会先寻找 Serializer。

```java
if (! Serializable.class.isAssignableFrom(cl)  
&& ! _isAllowNonSerializable) {  
	throw new IllegalStateException("Serialized class " + cl.getName() + " must implement java.io.Serializable");  
}
```

这里可以发现能够序列化非 Serializable 的子类

```java
if (_isEnableUnsafeSerializer  
	&& JavaSerializer.getWriteReplace(cl) == null) {  
	return UnsafeSerializer.create(cl);  
}  
else  
	return JavaSerializer.create(cl);
```

然后判断类中有没有 `writeReplace` 方法，如果有，那么便使用 JavaSerializer，否则使用 UnsafeSerializer。

一般是用 UnsafeSerializer Wrap。
跳过了 transient 和 static 修饰的字段
```java
if (Modifier.isTransient(field.getModifiers())  
|| Modifier.isStatic(field.getModifiers())) {  
	continue;
}
```

然后再给每个字段分配 Serializer
```java
private static FieldSerializer getFieldSerializer(Field field)
  {
    Class<?> type = field.getType();
    
    if (boolean.class.equals(type)) {
      return new BooleanFieldSerializer(field);
    }
    ...
    else
      return new ObjectFieldSerializer(field);
  }
```

## 触发点

`MapDeserializer#readMap` 对 Map 类型数据进行反序列化操作是会创建相应的 Map 对象，并将 Key 和 Value 分别反序列化后使用 put 方法写入数据。在没有指定 Map 的具体实现类时，将会默认使用 HashMap ，对于 SortedMap，将会使用 TreeMap。
- HashMap 在 put 键值对时，将会对 key 的 hashcode 进行校验查看是否有重复的 key 出现，这就将会调用 key 的 hasCode 方法
- TreeMap 在 put 时，由于要进行排序，所以要对 key 进行比较操作，将会调用 compare 方法，会调用 key 的 compareTo 方法
除此之外，还有一个能够利用的是 toString。可以在 expection 处理时通过字符串拼接调用 toString 方法。

限制：
- kick-off chain 起始方法只能为 hashCode/equals/compareTo 方法；
- 利用链中调用的成员变量不能为 transient 修饰；
- 所有的调用不依赖类中 readObject 的逻辑，也不依赖 getter/setter 的逻辑。

## 参考

- https://su18.org/post/hessian/