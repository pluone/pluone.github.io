---
layout: post
title:  "spring mvc multipart配置源码解析"
date:   2019-07-06 16:00:00 +0800
categories: 源码解析
---

## multipart配置的作用

multipart的配置主要有两个
maxFileSize 用于配置允许上传的单个文件的最大值
maxRequestSize 用于配置单个请求中允许上传的最大值，一个请求中可以上传多个文件
spring 2.x最新的配置为:

```yaml
spring.servlet.multipart.max-file-size: 10MB
spring.servlet.multipart.max-request-size: 100MB
```

`org.apache.catalina.connector.Request#parseParts`

## multipart配置如何起作用

从请求中解析上传的文件，解析的时候需要

```java
private void parseParts(boolean explicit) {
    MultipartConfigElement mce = getWrapper().getMultipartConfigElement();
    // ...
    ServletFileUpload upload = new ServletFileUpload();
    upload.setFileItemFactory(factory);
    upload.setFileSizeMax(mce.getMaxFileSize());
    upload.setSizeMax(mce.getMaxRequestSize());
    // ...
    List<FileItem> items = upload.parseRequest(new ServletRequestContext(this));
}

```

`org.apache.tomcat.util.http.fileupload.FileUploadBase#parseRequest`
`org.apache.tomcat.util.http.fileupload.util.LimitedInputStream#checkLimit`
返回的迭代器中的inputStrem是LimitedInputStream,在读取输入流的时候会检查读取字节的大小,如果超过了配置的上传文件大小则抛出异常。

### 配置是如何注入到tomcat容器中的呢

入口方法是
`org.apache.catalina.core.ApplicationServletRegistration#setMultipartConfig`
调用顺序则是
spring的自动配置会自动注册dispatcherServlet这个bean

1. `org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration.DispatcherServletRegistrationConfiguration#dispatcherServletRegistration`
2. `org.springframework.boot.autoconfigure.web.servlet.DispatcherServletRegistrationBean`
3. `org.springframework.boot.web.servlet.ServletRegistrationBean#configure`

```java
protected void configure(ServletRegistration.Dynamic registration) {
    super.configure(registration);
    String[] urlMapping = StringUtils.toStringArray(this.urlMappings);
    if (urlMapping.length == 0 && this.alwaysMapUrl) {
        urlMapping = DEFAULT_MAPPINGS;
    }
    if (!ObjectUtils.isEmpty(urlMapping)) {
        registration.addMapping(urlMapping);
    }
    registration.setLoadOnStartup(this.loadOnStartup);
    if (this.multipartConfig != null) {
        registration.setMultipartConfig(this.multipartConfig);
    }
}
```
