---
title: Eclipse使用Git
date: 2019-08-01 02:10:26
categories:
- 工具
tags:
top:
---

[TOC]

### 一、基本操作
![Git架构](https://img-blog.csdnimg.cn/20190421221809808.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpYW1vZnk=,size_16,color_FFFFFF,t_70)

1.工作空间 -> 本地仓库 : commit
2.本地仓库 -> 远端仓库 : push
3.远端仓库 -> 本地仓库 : fetch/clone
4.远端仓库 -> 工作空间 : pull

5.解决冲突 : add to index

6.本地仓库 -> 工作空间 : checkout

### 二、增加文件
新建文件 -> 右键 Team -> Add to index

### 三、解决冲突
#### 1.冲突查看
可以用Synchronize Workspace感知云端到工作空间的差异。不过不会做任何文件的修改或提交操作。

#### 2.冲突感知
必须要pull，工作空间才能感知到冲突(即显示冲突的红色箭头)。否则只是在push或直接pull时提示冲突，但还是无法感知。

#### 3.在冲突的情况下，几种操作场景示例
##### a.直接commit+push
结果：完成commit，代码卡在local仓库，提示冲突。但工作空间没有红箭头。
修复：pull + AddToIndex + commit + push
##### b.端直接pull
结果：完成fetch，代码卡在local仓库。工作空间显示红箭头。
修复：AddToIndex + commit + push
##### c.先commit, 再fetch, 再push
结果：commit完成，fetch完成，push报错。但是工作空间没有红箭头。
修复：pull + AddToIndex + commit + push
##### d.先fetch, 再commit, 再push
结果：fetch完成，commit完成，push报错。但是工作空间没有红箭头。
修复：pull + AddToIndex + commit + push

##### e.总结如下

* 本地仓库可以同时存在冲突，但是工作空间不会提示红色箭头，push时则报错。
* 只有pull时，工作空间才会提示红箭头。
* 可以通过sync查看冲突，但也仅仅是查看，不会做任何改变，工作空间也不会提示红箭头。
* 其实是因为，fetch的时候，只是检查了remote端仓库的最新版本，并没有把remote的代码搞过来。所以，本地仓库是允许commit和fetch的代码，在有冲突的情况下共存的。

解决冲突时，记得选UseHEADofConfictingFiles。使用本地文件。因为HEAD是指向本地的最小文件。然后origin则一般用来指代远程服务器。


### 四、新建分支
push branch "master"...选项可提供新建分支的选项。
swich to 也提供新建分支的选项。
Push/Fetch to upstream则是默认操作当前分支。


### 五、回滚操作
直接showInHistory -> checkOut。


### 六、其他
记得配置ignore文件，免得把.project，.setting等文件提交了。