---
title: gitflow规范文档
date: 2020-10-31 20:17:03
tags:
	- Android
categories: 文档
---

本规范引用自[A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/)

<!--more-->

下图是Git Flow的基本流程图：

<img src="https://nvie.com/img/git-model@2x.png" style="zoom:60%;" />

#### 分布式开发

每个人都对origin库拉代码和提交代码，每个人在开发过程中可以从子团队的其他队友那里获得代码版本变更，以最终向origin库发起一个整体的pull request。

#### 分支

##### 主分支

<img src="https://nvie.com/img/main-branches@2x.png" alt="主要分支" style="zoom:60%;" />

###### master

+ 命名：master-xxx
+ 含义：发布的代码版本，在此分支上打tag来区分版本
+ 注意：一般主分支有且仅有一个不可以更改只可以从其他分支合并，当应用有多个不同版本（例如内部结构不同的两个项目版本）时，可以根据版本名称或特性添加后缀名

###### develop

+ 命名：develop-xxx
+ 含义：在此类分支上进行主要的开发
+ 注意：开发合并和fork主要以此类分支为准，要注意经常合并更新

##### 辅助性分支

###### feature

+ 命名：feature-xxx
+ 含义：在此类支上进行特定需求或功能的开发
+ 注意：从相关的develop分支出功能分支，开发完成后要及时合并到develop分支上并进行丢弃删除
  + 使用`git merge --no-ff xxx`命令来进行分支合并，--no-ff标志导致合并操作创建一个新的commit对象，即使该合并操作可以fast-forward。这避免了丢失这个功能分支存在的历史信息，将该功能的所有提交组合在一起。

###### release

+ 命名：realease-vn.n.n(版本号)
+ 含义：为新版本发布做准备所使用的，允许我们在最后做一些修改（如发布前的测试、调试发现的bug修改工作），
+ 注意：
  + 从阶段开发完成的develop分支中分离出来的，最终要合并到develop和master分支上。
  + 所有后续版本要添加的feature必须要等到release分支创建以后再进行合并。
  + 真正发行版本的时候，要先把release分支合并到master分支上，并打上版本tag。
  + 完成对develop分支和master分支的合并之后，删除当前的release分支

###### hotfix

<img src="http://static.oschina.net/uploads/img/201302/25142848_NuIv.png" style="zoom:80%;" />

+ 命名：hotfix-vn.n.n(版本号)
+ 含义：与release分支很相似，但是hotfix分支是未经计划的版本更新，可能是由于异常状态或者是某些马上需要修复的缺陷
+ 注意：
  + 完成一个bugfix之后，需要把butfix合并到master(打tag)和develop分支去
  +  如果一个release分支已经存在，那么应该把hotfix合并到这个release分支，而不是合并到develop分支
