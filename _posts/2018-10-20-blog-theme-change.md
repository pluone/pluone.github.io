---
layout: post
title:  "博客主题迁移"
date:   2018-10-20 14:40:00 +0800
categories: 杂项
excerpt_separator: <!--more-->
---

Jekyll的默认主题是minima，对中文的支持不够友好，界面很丑，在网上找了一款新的主题NexT，发现显示效果还不错，于是决定将minima主题迁移为NexT主题。主题迁移的过程中希望保留原仓库提交历史，迁移的过程中走了一些弯路，
下面是迁移完成之后总结出来最简单的迁移过程。
<!--more-->
过程如下：

1. 更改原博客仓库pluone.github.com，只保留_posts目录和assets目录，其他都是跟原主题有关系的文件，全部删除，然后commit
2. 克隆新主题仓库，把所有文件都拷贝到原仓库，合并_posts和assets目录
3. 进行提交代码即可。
