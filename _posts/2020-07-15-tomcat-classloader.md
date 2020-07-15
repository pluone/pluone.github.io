---
layout: post
title:  "tomcat的类加载机制"
date:   2020-07-15 17:00:00 +0800
categories: 后端开发
---


首先回答一个问题，tomcat为什么要设计自己的类加载机制？

> Java标准的类加载器及其双亲委派机制是`AppClassLoader -> ExtClassLoader -> BootstrapClassLoader`,
其中`AppClassLoader`是系统默认的类加载器

我们用反证法来解答一下这个问题。tomcat是一个servlet容器，加载servlet类需要一个类加载器，
如果不用自己的类加载器，那么会使用默认的`AppClassLoader`，但是使用
`AppClassLoader`会带来一些问题，具体说来：

因为一个tomcat下可以同时部署多个项目（或者说应用），项目于项目之间的servlet类
需要互相隔离开，如果使用AppClassLoader则所有项目共享相同的类路径（CLASSPATH），项目于项目的源码之间
无法实现隔离，可能造成严重的安全问题，因此tomcat需要解决这个问题，

同时为了支持reloadable，reloadable是这样一项技术，如果项目下WEB-INF/classes 或者 WEB-INF/lib目录下的文件有改动，
即可自动重新加载(reload)项目，在项目开发阶段会特别有用。

为此tomcat设计了自己的类加载机制，顶层的抽象接口是`Loader`接口。

### 顶层设计-Loader接口

```java
public interface Loader {
    void backgroundProcess();

    ClassLoader getClassLoader();

    Context getContext();

    void setContext(Context var1);

    boolean getDelegate();

    void setDelegate(boolean var1);

    boolean getReloadable();

    void setReloadable(boolean var1);

    void addPropertyChangeListener(PropertyChangeListener var1);

    boolean modified();

    void removePropertyChangeListener(PropertyChangeListener var1);
}
```


Loader接口中通过getClassLoader来获取到自定义的类加载器，同时定义了其它一些方法，具体方法的作用如下：

`getClassLoader`方法可以获取到自定义的类加载器，获取到的类加载器是`WebappClassLoaderBase`的子类  
`getContext`和`setContext` 用来将`Loader`关联到`Context`上，`Context`代表了一个项目（或者说应用），其标准实现类是`StandardContext`，因此不同的项目有不同的`Loader`，从而实现了项目于项目之间源码的隔离。  
`getDelegate`和`setDelegate` 用来设置启用/关闭标准的双亲委派机制，设置为true则不再使用自定义的类加载器，而是使用系统默认AppClassLoader。  
`getReloadable`和`setReloadable` 用来设置启用/关闭reloadable, 如果启用reloadable则会通过backgroundProcess启动的后台线程自动监测项目WEB-INF/classes和WEB-INF/lib目录下文件的变动，如果发生变动，会自动reload项目。  
`modified`方法则用来配合reloadable，发生变动才会reload。  


### 自定义类加载器-WebappClassLoader

具体的类加载任务是由`WebappClassLoader`来做的，这个类的父类是`WebappClassLoaderBase`，核心的代码是在父类中的，加载类的入口方法是loadClass，加载的具体的流程如下：

1. 查找本地缓存，如果在缓存中找到了已经加载过的类则直接返回
2. 查找系统ClassLoader的内部缓存，如果加载过直接返回
3. 使用ExtClassLoader加载类
4. 调用findClass方法加载类，findClass是WebappClassLoaderBase中自定义的方法，一般是去war包下的路径中查找类
5. 如果仍然找不到，委托给WebappClassLoader的parent去查找，这里的parent一般为URLClassLoader

需要注意的是第3步到第4步绕过了AppClassLoader，即破坏了默认的双亲委派机制。


具体的代码摘录如下，删除冗余部分
```java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

        // 加锁
        synchronized (getClassLoadingLock(name)) {
            Class<?> clazz = null;

            // step1 检查缓存，这里的缓存是自定义的缓存，看是否已经加载过这个类，如果加载过直接返回
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }

            // step2 检查系统的缓存，看是否已经加载过这个类
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }

            // step3 获取系统类加载器，这里获取到的是ExtClassLoader,不是AppClassLoader,因此绕过了系统默认的双亲委派机制
            ClassLoader javaseLoader = getJavaseClassLoader();
            try {
                clazz = javaseLoader.loadClass(name);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }

            // step4 调用findClass查找类，这里是tomcat自己实现的类加载，从项目下WEB-INF/classes和WEB-INF/lib这两个目录去加载servlet需要的类
            try {
                clazz = findClass(name);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from local repository");
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }

            // step5 还是找不到的话，委托给父类加载器去加载
            if (!delegateLoad) {
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }
        }

        throw new ClassNotFoundException(name);
    }
```