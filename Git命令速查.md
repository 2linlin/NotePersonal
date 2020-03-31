---
title: Git命令速查
date: 2019-08-01 02:16:26
categories:
- 工具
tags:
top:
---

#### 1.创建git仓库

```shell
git init --bare blog.git
```

#### 2.克隆远端库到本文件夹，使用git用户登录

```shell
git clone git@服务器IP:/data/wzh-blog/git-repo/blog.git
```

###### 本地代码提交

```shell
git add .
git commit -m 'test'
git push
```

###### 远程代码拉取

```shell
git pull
```

###### 修改用户名和邮箱

```shell
git config user.name "git"

git config user.email "wuzeheng@hotmail.com"
```

#### 3.将本地新项目推送至git

```shell
git init
git add .
git commit -m "first commit"
git remote add origin https://github.com/h2linlin/spring-boot-web-demo.git
git push -u origin master
```

#### 4.把本地代码上传（更新）到github上

```shell
git init 						// 建立git仓库
git add test.txt				// 加入版本管理
git commit -m "第一次提交"		 // 提交
git remote add origin git@github.com:xxx/xxxx.git	//本地仓库关联到github上
git pull --rebase origin master	// pull并合并到本地代码
git push -u origin master		// push到远端
```



#### 5.从GitLab上拉新项目

Git 全局设置
```shell
git config --global user.name "Administrator" git config --global user.email "admin@example.com" 
```
创建新版本库
```shell
git clone http://dee063db4600/cdm/spring-boot-demo.git cd spring-boot-demo touch README.md git add README.md git commit -m "add README" git push -u origin master
```
已存在的文件夹
```shell
cd existing_folder
git init
git remote add origin http://dee063db4600/cdm/spring-boot-demo.git # 重点
git add .
git commit -m "Initial commit"
git push -u origin master # 重点
```
已存在的 Git 版本库

```shell
cd existing_repo git remote rename origin old-origin git remote add origin http://dee063db4600/cdm/spring-boot-demo.git git push -u origin --all git push -u origin --tags
```

#### 6.合并dev到master

1.获取并检出此合并请求的分支

```shell
git fetch origin
git checkout -b dev origin/dev
```

2.本地审查变更

3.合并分支并修复出现的任何冲突

```shell
git checkout master
git merge --no-ff dev
```

4.推送合并的结果到 GitLab

```shell
git push origin master
```



### create a new repository on the command line

```shell
echo "# cdm-web-demo" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/h2linlin/cdm-web-demo.git
git push -u origin master
```



###  push an existing repository from the command line

```shell
git remote add origin https://github.com/h2linlin/cdm-web-demo.git
git push -u origin master
```