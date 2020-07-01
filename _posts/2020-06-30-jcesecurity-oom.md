---
layout: post
title:  "记一次内存溢出排查问题的过程"
date:   2020-06-30 10:00:00 +0800
categories: 后端开发
---

问题背景：最近开发了一个调用第三方支付平台进行代付的功能，部署到测试环境进行测试，经过压力测试发现代码无法完成功能，并且出现了响应缓慢的情况。
在发现问题后，我们观察了日志，发现请求日志打印了一半，之后就没有了，并且发现了一处其它地方的内存异常。

```
java.util.concurrent.ExecutionException: java.lang.OutOfMemoryError: GC overhead limit exceeded
```

这里我们执行业务的一段代码如下：
```java
// 异步执行付款
public void asyncExecutePay(List<PayInfo> payList) {
    Runnable runnable = () -> {
        try {
            for(PayInfo payInfo: payList){
                // 业务逻辑
            }
        }catch (Exception e) {
            logger.error("执行代付异常", e);
        }

    };
    CompletableFuture.runAsync(runnable);
}

```

此时合理的解释是程序执行到一半，发生了内存溢出，因此日志也只有一半，而且因为catch的是`Exception`，`OutOfMemoryError`并非是`Exception`的子类,所以代码里没有找到这段业务的错误日志，而内存的不足引起了其它地方的溢出。

问题很明显，接下来我们进一步定位问题

### 定位问题

使用jstat工具查看gc的相关信息

- 查看gc信息

```
jstat -gc 19010
```

结果如图
![oom1](/assets/images/oom/oomm1.png)


- 查看gc使用率信息
```
jstat -gcutil 19010
```

结果如图
![oom2](/assets/images/oom/oomm2.png)

- 查看gc容量信息

```
jstat -gccapacity 19010
```

结果如图
![oom3](/assets/images/oom/oomm3.png)

### 生成内存转储文件

通过这些信息，发现新生代和老年代内存的占用率均达到了99%以上，并且Full GC的次数和时间非常夸张。

由于JVM启动时没有指定`HeapDumpOnOutOfMemoryError`，所以需要先手动生成内存转储文件

```
jmap -dump:live,format=b,file=/tmp/19010dump.hprof 19010
```

执行完成后发现dump文件竟然达到惊人的5.9G, 从服务器下载到本地进行分析

分析工具一开始用的是jvisualvm，这个工具一上来加载转储文件时就内存溢出了，还没分析别人自己先溢出了😅

```
out of memory in heap walker： To avoid this error，increase the -Xmx value in the etc/netbeans.conf file in NetBeans IDE installation directory.
```

查询了一下可以启动时指定参数如下

```
jvisualvm -J-Xms1024m -J-Xmx2048m
```

之后使用的过程中感觉加载很慢，就换了另外一个工具来分析。

### 使用Eclipse MAT(Memory Analyzer Tool)分析 

启动时同样由于文件实在是太大使用时报错，查询了一下启动时可以如下指定Xmx

```
/Applications/mat.app/Contents/MacOS/MemoryAnalyzer -vmargs -Xmx4g
```
![mat1](/assets/images/oom/mat1.png)

可以看到` javax.crypto.JceSecurity`对象占用了3.5G的空间


然后点击List objects -> withoutging references查看这个对象内部情况
可以看到属性`verificationResults`占用了3.5G左右的空间

![mat2](/assets/images/oom/mat2.png)

下面给`verificationResults`一个特写，可以看到是内部的table数组占用的空间，而且数组大小是6420，这里只展开了25项，全部展开后可以发现
数组对象中都是`org.bouncycastle.jce.provider.BouncyCastleProvider`，而且每项的空间占用591368字节，大约0.5M，共6420项，总和刚好是3.5G左右。

![mat3](/assets/images/oom/mat3.png)

`javax.crypto.JceSecurity`是干嘛用的呢`verificationResults`字段又是做啥的，需要研究一下源码

```java
// 代码片段摘录
final class JceSecurity {
    private static final Map<Provider, Object> verificationResults = new IdentityHashMap();

    // 简单看了一下这段代码用来对Provider的源代码jar文件进行安全验证，然后将验证结果存为true或者false
    static synchronized Exception getVerificationResult(Provider var0) {
        // 这里一上来从map中获取验证结果，存在直接返回
        Object var1 = verificationResults.get(var0);
        if (var1 == PROVIDER_VERIFIED) {
            return null;
        } else if (var1 != null) {
            return (Exception)var1;
        } else if (verifyingProviders.get(var0) != null) {
            return new NoSuchProviderException("Recursion during verification");
        } else {
            Exception var3;
            try {
                verifyingProviders.put(var0, Boolean.FALSE);
                URL var2 = getCodeBase(var0.getClass());
                verifyProviderJar(var2);
                // 这里验证成功了，存为true
                verificationResults.put(var0, PROVIDER_VERIFIED);
                var3 = null;
                return var3;
            } catch (Exception var7) {
                // 这里验证失败了，存为异常信息
                verificationResults.put(var0, var7);
                var3 = var7;
            } finally {
                verifyingProviders.remove(var0);
            }

            return var3;
        }
    }
}
```

通过阅读代码可以理解这段代码大致是在验证Provider的源码是否安全，验证成功将结果放进map中，key即类名，value是true或false。

这里一开始先从map中找结果，如果是已经验证成功的类会直接返回。

看到这里你一定会有一个疑问map中的key不是唯一的吗，为什么这个map有还是有6420个一个的key呢，
事实上需要注意到这里map的类型是`IdentityHashMap`，这个map不同于常用的`HashMap`，它判断key是否相等的条件是`k1==k2`,`HashMap`判断key相等的条件是
`k1==k2 || k1.equals(k2)`，也就是说`IdentityHashMap`判断对象相等的条件是严格的内存地址相等，同类的不同实例是不满足条件的。

至此，原因被找到了，每次传进来的都是`BouncyCastleProvider`类的不同实例

### 定位业务逻辑代码问题

所以接下来需要定位到业务代码哪里调用的

因此我们本地启动项目进行Debug, 在JceSecurity中打断点，触发业务逻辑，程序在断点处停下来，然后可以看到调用的堆栈，可以分析代码是如何一步一步执行到JceSecurity中的。

调用堆栈信息如下图

![stack](/assets/images/oom/code-stack.png)

最终发现是`KeyStore.load(fiKeyFile, pass.toCharArray());`这样一行代码触发的，经过深入调查发现这里的Provider使用了`BouncyCastleProvider`, 而每次获取一个新的`KeyStore`实例，都会往`javax.crypto.JceSecurity#verificationResults`中存放一次`BouncyCastleProvider`，因此造成了内存溢出。

触发的业务逻辑代码段如下：

```java
/**
*   这是一段签名算法的代码，使用p12私钥文件，给字符串签名，返回签名结果
*   msg 待签名字符串
*   keyfile 私钥文件路径
*   pass 私钥文件密码
*/
public String signMsg(String msg, String keyfile, String pass) throws AIPGException {
    String signedStr = null;
    try {
        KeyStore ks = prvd == null ? KeyStore.getInstance("PKCS12") : KeyStore.getInstance("PKCS12", prvd);
        try (FileInputStream fiKeyFile = new FileInputStream(keyfile)) {
            // 这一行代码执行后调用链会一直到达JceSecurity这个类里面，最终导致了内存溢出
            ks.load(fiKeyFile, pass.toCharArray());
        } catch (Exception ex) {
            
        }
        Enumeration<String> aliases = ks.aliases();
        String keyAlias = null;
        RSAPrivateCrtKey prikey = null;
        while (aliases.hasMoreElements()) {
            keyAlias = (String) aliases.nextElement();
            if (ks.isKeyEntry(keyAlias)) {
                prikey = (RSAPrivateCrtKey) ks.getKey(keyAlias, pass.toCharArray());
                break;
            }
        }

        Signature sign = prvd == null ? Signature.getInstance("SHA1withRSA") : Signature.getInstance("SHA1withRSA", prvd);
        sign.initSign(prikey);
        sign.update(msg.getBytes(encoding));
        byte[] signed = sign.sign();
        byte[] sign_asc = new byte[signed.length * 2];
        Hex2Ascii(signed.length, signed, sign_asc);
        signedStr = new String(sign_asc);
    } catch (Exception e) {
    }
    return signedStr;
}
```

解决方法

看到一个方案是使用`Security.addProvider(new BouncyCastleProvider())`使用这样一行代码，后来发现会有新的问题，
最终放弃使用BouncyCastleProvider后问题消失。


参考:  

https://www.dazhuanlan.com/2020/01/30/5e322756695f7/  
https://my.oschina.net/u/867417/blog/828199  
https://bugs.openjdk.java.net/browse/JDK-8168469

