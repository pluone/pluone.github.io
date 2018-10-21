---
layout: post
title:  "gitflow工作流程"
date:   2018-09-03 12:00:00 +0800
categories: 后端开发
---

gitflow是使用git协作的一个工作流程,是经由Vincent Driessen at nvie提出和发扬光大的.  
gitflow主要围绕着项目上线定义了一些严格的分支模型,为管理大型项目提供了一个健壮的框架.

gitflow只是一个概念模型,git-flow toolset则为这种概念的应用提供了具体的git命令扩展.

## git-flow toolset的安装

MAC OS系统下使用命令`brew install git-flow`
window下安装的git已经集成了git-flow toolset

## develop和master分支

一般的git项目初始话之后就只有一个master分支,gitflow则同时使用develop分支来记录项目历史,其中
master分支用于记录上线历史
develop则作为feature分支的集成分支

![master-develop分支图](/assets/gitflow/master-develop.svg)

使用这个命令来创建一个新的develop分支

```git
git branch develop
git push -u origin develop
```

如果使用gitflow toolset则可以使用git flow init来达到相同的效果

```git
$ git flow init
Initialized empty Git repository in ~/project/.git/
No branches exist yet. Base branches must be created now.
Branch name for production releases: [master]
Branch name for "next release" development: [develop]

How to name your supporting branch prefixes?
Feature branches? [feature/]
Release branches? [release/]
Hotfix branches? [hotfix/]
Support branches? [support/]
Version tag prefix? []

$ git branch
* develop
 master
```

## feature分支

feature分支使用develop分支作为其父分支
![feature分支图](/assets/gitflow/feature.svg)

### feature分支的创建

```git
git checkout develop
git checkout -b feature_branch
```

等同于

```git
git flow feature start feature_branch
```

### 完成feature分支

```git
git checkout develop
git merge feature_branch
```

等同于

```git
git flow feature finish feature_branch
```

## release分支

一旦develop分支获得了足够的feature分支可以用来发布之后,我们从develop分支上fork出一个release分支,此时便可以进行下一次上线迭代,新的feature分支可以从这个地方开始创建.
release分支则用于bug修复,文档生成和其它与这次上线有关的任务.一旦可以开始上线,release分支会被合并到master分支上并且打标签,同时release也要被合并到develop分支上去
![release分支图](/assets/gitflow/release.svg)

### release分支的创建

```git
git checkout develop
git checkout -b release/0.1.0
```

等同于

```git
$ git flow release start 0.1.0
Switched to a new branch 'release/0.1.0'
```

### 完成release分支

```git
git checkout develop
git merge release/0.1.0
git checkout master
git checkout merge release/0.1.0
```

等同于

```git
git flow release finish '0.1.0'
```

## hotfix分支

hotfix分支用于为生产环境打补丁
![hotfix分支图](/assets/gitflow/hotfix.svg)

### hotfix分支创建

```git
git checkout master
git checkout -b hotfix_branch
```

等同于

```git
git flow hotfix start hotfix_branch
```

### 完成hotfix分支

```git
git checkout master
git merge hotfix_branch
git checkout develop
git merge hotfix_branch
git branch -D hotfix_branch
```

等同于

```git
git flow hotfix finish hotfix_branch
```

## 总结

gitflow的整体流程如下:

1. 从master分支上创建develop分支
2. 从develop分支上创建feature分支
3. 当feature分支完成后被合并到develop分支上
4. 从develop分支上创建release分支
5. 当release分支完成后被合并到master和develop上
6. 生产环境上发现问题时从master上创建hotfix分支
7. hotfix完成后被合并到master和develop上

网络上关于gitflow的文章很多,亦可参见:  
[A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/)  
[git-flow 备忘清单](https://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)

参考:  
全文翻译自,做了适当修改  
<https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow>