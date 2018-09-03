---
layout: post
title:  "H5页面mixed-content错误"
date:   2018-09-03 18:00:00 +0800
categories: java
---

## 问题

前端H5页面调用后端接口后页面一直是白屏状态,没有任何响应.

前端H5页面通过JS调用后端的一个接口,接口返回字符串类型数据,内容是一个html表单,如下(内容进行了打码和格式化处理):

```html
<form id='myForm' action='http://openapi-test.xxx.com/api/callback/h5' method='post'>
    <input type='hidden' name='serviceName' value='xxx' />
    <input type='hidden' name='platformNo' value='xxx' />
    <input type='hidden' name='userDevice' value='MOBILE' />
    <input type='hidden' name='reqData' value='xxx'/>
    <input type='hidden' name='keySerial' value='xxx' />
    <input type='hidden' name='sign' value='xxx'/>
</form>
<script type='text/javascript'>document.getElementById('myForm').submit();</script>
```

这是一个可以自动提交的表单.ngnix没有收到访问日志.

## 问题排查

通过chrome的开发者工具,模拟操作后发现js报错如下:
![mixed content error](/assets/mixed-content-error.png)
查询之后才恍然大悟,原来我们这个H5页面是https协议的,但是这个表单提交的地址是http协议的,于是被浏览器认为是不安全的被阻止掉了,所以页面一直是空白.

## 经验总结

发生这个问题后我们绕了好久才开始使用chrome来模拟请求,进行debug.
其实关于H5页面的问题,多数情况下应该是js错误引起的,所以要有打开控制台debug的意识.