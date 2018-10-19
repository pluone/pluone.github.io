---
layout: post
title:  "由java system property引起的问题及源码分析"
date:   2018-08-08 21:00:00 +0800
categories: 框架
---

## 问题的发生过程是这样的

我们有一个sdk项目,假定名称为alibaba-sdk,封装了一些第三方的接口进去,其中的配置有很多,所以把配置文件都放在项目下的properties文件中,分别为`test.properties`和`production.properties`,希望能够在测试环境加载test中的配置,在生产环境下加载producction中的配置.
那么作为一个纯java项目如何判断当前环境呢,因为alibaba-sdk是我们内部用的,并且都是被spring项目依赖,所以想到了使用`Sytem.getProperty("spring.profiles.active")`这种方式来读取当前profile从而实现根据不同的profile加载不同的配置,源码如下:

```java
@Slf4j
public class Config {
    private static Properties properties;

    static {
        properties = new Properties();
        String activeProfile = System.getProperty("spring.profiles.active");
        log.info("current profile is:{}", activeProfile);
        String fileName = "/test.properties";
        if ("production".equals(activeProfile)) {
            fileName = "/production.properties";
        }

        try {
            properties.load(new InputStreamReader(Config.class.getResourceAsStream(fileName), "UTF-8"));
        } catch (IOException e) {
            log.error(e.getMessage(), e);
        }
    }


    public static String getProperty(String property) {
        return properties.getProperty(property);
    }
}
```

测试环境很ok,结果到了生产环境却出了问题,生产环境读到的配置竟然是test的配置,这有点反常.

## 下面是排查过程

第一时间我们加了这一行代码`log.info("current profile is:{}", activeProfile);`(这里可以体现日志的重要性,写代码的好习惯是关键的地方都要加上日志),然后再次部署发现日志里打出的竟然是null,没有读取到profile,想来想去觉得应该看一下项目是如何部署的,于是我们查了一下`.gitlab-ci.yml`文件,这里面描述了部署时的操作,  
`bash ci/docker.sh -e production -p asset-m`  
docker.sh文件中有一行是这样的  
`docker -H ${BUILD_HOST} build -t ${IMAGE_NAME} --build-arg SPRING_PROFILE_ACTIVE=${SPRING_ENV} --build-arg PROJECT_BUILD_FINALNAME=${PROJECT_BUILD_FINALNAME} ${BUILD_PROJECT}`  
其中production被传递给了SPRING_PROFILE_ACTIVE这个参数,然后Dockerfile是这样的  
`CMD ["bash","-c","java -jar /${PROJECT_BUILD_FINALNAME}.jar --spring.profiles.active=${SPRING_PROFILE_ACTIVE}"]`  
也就是说最终通过`--spring.profiles.active=production`这样的方式完成profile的传递,这个杠杠参数传递是个什么鬼,没有见过这种用法啊,因为我发现本地debug的时候项目都是通过这样的方式启动的  
`/usr/local/Cellar/adoptopenjdk-openjdk8/jdk8u172-b11/bin/java -Dspring.profiles.active=dev -Dspring.output.ansi.enabled=always com.xxx.xxx.Application`  
-D的方式设置的profile,明显这两种方式都可以设置profile,但是杠杠的这种方式设置的不是java system property,所以也就无法通过`Sytem.getProperty("spring.profiles.active")`的方式来读取.

修改后的Dockerfile变成了这样,然后问题得到了解决.  
`CMD ["bash","-c","java -Dspring.profiles.active=${SPRING_PROFILE_ACTIVE} -jar /${PROJECT_BUILD_FINALNAME}.jar"]`

## 总结一下

-D的方式来设置system property这是标准的做法,可以通过`Sytem.getProperty("spring.profiles.active")`这种方式读取到设置的property  
`--spring.profiles.active`这个不是java的做法,是Spring提供的一种读取参数的用法,称为spring command line argument,其实杠杠后面的所有参数都被认为是main函数中的args参数,然后Spring进行了解析处理,后面有详细的源码分析.

### 非技术总结

Dockerfile文件看起来有点吃力,这个是我的一个短板

## 如果你愿意多看一下,下面是源码分析

## 问题一:Spring command line argument是如何被解析的

`org.springframework.boot.SpringApplication#configureEnvironment`的代码

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

```java
//注意这里的args参数,就是main函数的args参数
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            //args参数被SimpleCommandLinePropertySource包装了一下
            composite.addPropertySource(new SimpleCommandLinePropertySource(
                    name + "-" + args.hashCode(), args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
```

一层层往里走看到这里就是解析杠杠参数的地方
`org.springframework.core.env.SimpleCommandLineArgsParser#parse`

### 问题二:Spring是如何读取system property中的profile的

这个就是`org.springframework.boot.SpringApplication#configureProfiles`做的事情了

```java
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    //源码的第一行就是在这里读取了system property
    environment.getActiveProfiles(); // ensure they are initialized
    // But these ones should go first (last wins in a property key clash)
    Set<String> profiles = new LinkedHashSet<String>(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(profiles.toArray(new String[profiles.size()]));
}
```
一直沿着引用链往里走发现最终的代码到了这里`org.springframework.core.env.AbstractEnvironment#doGetActiveProfiles`,
这段代码其实去system property读取了"spring.profiles.active"这个属性.