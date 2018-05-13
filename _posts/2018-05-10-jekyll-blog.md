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
