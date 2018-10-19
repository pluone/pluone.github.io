---
layout: post
title:  "如何用Jekyll写blog?"
date:   2018-05-10 18:00:00 +0800
categories: jekyll update
---

jekyll的blog存放在`_posts`目录下,文件名格式是`YEAR-MONTH-DAY-title.md`,其中年必须是4位数字,月和日必须是2位数字

开头必须是 YAML Front Matter,可以理解为博客内容元信息,之后才是内容.

### 引入图片和其它资源

一般在jekyll根目录下创建asset目录,将资源文件放在该目录下

在本地预览博客内容
`bundle exec jekyll serve`
后打开`http://127.0.0.1:4000/`

### 配置文件

```yml
#默认的主题是minima,博文的日期默认是英文日期格式,可以自定义为`yyyy-MM-dd`的日期格式
minima:
  date_format: "%Y-%m-%d"

#首页显示博文摘录
show_excerpts: true

#启用disqus评论功能,只需要指定shortname即可,默认在生产环境下才会生效
#需要注册disqus账号,按照disqus官网的提示需要在代码中嵌套js才能开启评论功能,不过所幸minima主题默认已经做了配置,这里只需要指定shortname
#默认的评论功能是开启的,可以通过在每篇博文的YAML Front Matter上使用`comments : false`关闭评论功能
disqus:
  shortname: pluone

#开启Google Analytics,生产环境下才生效
google_analytics: UA-NNNNNNNN-N
```

### 使用vscode配合markdownlint语法检查

需要自定义一些markdownlint的配置项,需要修改vscode的配置文件,添加如下项:

```json
 "markdownlint.config": {
        "MD013": false,
        "MD002": false, //禁用文章开头必须为H1标题栏
        "MD033": false, //disable no-inline-html
        "MD041": false, //disable first-line-h1
  }
```

### 关于博客主题

当我们使用jekyll new <blog_name> 创建一个博客站点的时候,默认使用的主题是minima,
可以使用`bundle show minima`来查看主题的目录位置,使用`open $(bundle show minima)`来打开这个目录可以看到主题的目录结构

参考:  
<https://github.com/jekyll/minima>  
<https://github.com/stidio/stidio.github.io>