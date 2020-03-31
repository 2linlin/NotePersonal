---
title: Hexo+Git+CentOS搭建博客
date: 2019-07-30 01:41:48
categories:
- 工具
tags:
top:
---



[TOC]

总体思路是这样的：

1. 本地Git提交md代码到服务器Git。
2. 服务器Git触发钩子执行shell脚本：
   1. 将提交过来的md代码检出到hexo的source目录。
   2. hexo根据source目录的md代码生成整个静态网站html文件，输出到hexo的public目录。
   3. 将hexo的public目录内容拷贝到网站部署目录下。(为了部署和开发文件分离，所以没直接用public目录)
3. 用Nginx把网站的访问指向网站部署目录就OK了。

实际的操作体验：

1. 用户在本地同步服务端的git。
2. 用户在本地编辑md文件，用git提交到服务器。
3. 服务器自动触发脚本，自动更新网站。

Hexo原理参考(了解下即可)：

​	Hexo其实就是一个“md转html的文件转换器”而已。Hexo主要有三个文件夹，source，theme，public。source存的是素材，主要就是md文件和图片等。theme存的是主题，规定了怎么组合source中素材来生成最终的网页。然后结合source和theme就可以生成一整个网站的静态html文件，存到了public里。所以public存的就是一整个生成的静态网站。

这篇博客主要基于[这篇文章](https://www.laoyuyu.me/2017/10/10/hexo_deploy_vps/) 的基础上进行改进，以及使用了[这个Hexo主题](<https://yelog.org/>)。然后自己一路踩坑过来。用到的命令，除了特别说明，都是root操作。

### 一、环境安装

#### 1.1 node js 安装

```shell
yum install gcc-c++ make
yum -y install nodejs
```

验证：

```shell
node -v 
npm -v
```

#### 1.2 安装git、nginx

Git 用于版本管理和部署，Nginx 用于静态博客托管。

``` shell
yum install git nginx -y
```

#### 1.3 安装hexo

* 我们使用 Node.js 的包管理器 npm 安装 hexo-cli 和 hexo-server

``` shell
npm install hexo-cli hexo-server -g
```

hexo-cli 是 Hexo 的命令行工具，可用于快速新建、发布、部署博客；

hexo-server 是 Hexo 的内建服务器，可用于部署前的预览和测试。-g 选项，表示全局安装。

* 验证

``` shell
hexo
# 如果报错如下，那么是因为需要更新nodejs至最新的10版本
/usr/lib/node_modules/hexo-cli/node_modules/chokidar/index.js:150
```

验证报错解决方法：

```shell
# yum upgrade会无效，因为老源还是6版本。需要清空缓存，获取最新源。
# 引用自：https://blog.mutoe.com/2018/upgrade-nodejs-using-yum/
# 无需删除旧版本的 nodejs，该问题只与 yum 缓存有关

# 以下命令请均以 sudo 权限执行

yum clean all # 清空 yum 缓存
rm -rf /var/cache/yum/* # 手动删除所有 yum 缓存
cd /etc/yum.repos.d/
ls
# 请结合实际情况删除所有以 node 开头的 repo 文件
rm -f nodesource-el.repo # 移除旧的 nodesource 源
curl -sL https://rpm.nodesource.com/setup_10.x | bash - # 安装 nodesource 源
yum update nodejs # 升级到最新版本
node # 验证
```

如果版本还是原来的(版本未升级成功)，那么删掉nodejs，重新装一次：

```shell
yum remove nodejs -y;
yum install nodejs -y;
# 重新安装完验证OK
```



### 二、git配置

#### 2.1 在云服务器上创建一个 GIT 用户，用来运行 GIT 服务

- 创建用户：`adduser git`
- 设置密码：`passwd git`

#### 2.2 创建证书 （如果用账号密码登录的话，这一步不用执行）

- 切换到git用户：`su git`
- 创建.ssh目录：`mkdir .ssh && chmod 700 .ssh`
- 然后在云服务创建`authorized_keys`公钥保存文件：`touch .ssh/authorized_keys && chmod 600 .ssh/authorized_keys`
  **tip:** 公钥保存文件`authorized_keys`是一行添加一个

#### 2.3 创建git仓库目录

创建一个名为git-repo的git仓库。下面的操作还同时多建了一个wzh-blog的父目录，用来装所有博客相关的文件。

``` shell
mkdir /data/wzh-blog/git-repo
cd /data/wzh-blog/git-repo
git init --bare blog.git
```

#### 2.4 配置 GIT HOOKS

**这里就是整篇文章的关键**，配置git仓库为：如果有md代码提交，就自动检出到Hexo目录，然后Hexo自动生成网站文件，然后自动发布到Nginx网站目录，这样网站内容就自动更新了。

下面这个post-receive文件，如果没有的话自己新建：

``` shell
vim /data/wzh-blog/git-repo/blog.git/hooks/post-receive
```

添加

``` shell
#!/bin/sh
git --work-tree=/data/wzh-blog/hexo/source/_posts --git-dir=/data/wzh-blog/git-repo/blog.git checkout -f;
cd /data/wzh-blog/hexo;
hexo clean;
hexo g;
rm -rf /data/wzh-blog/www/*; cp -r /data/wzh-blog/hexo/public/* /data/wzh-blog/www;
```

后续如果有目录问题，可以看下上述脚本，是不是你创建目录的时候创建错了。另外上述脚本中有 rm -rf，执行的时候小心点，千万要确保目录是上述目录，不要因为脚本目录写错导致误删除。

#### 2.5 为了安全考虑，禁用GIT用户的SHELL 登录权限配置

这样git用户就只能用来执行脚本，无法切换到shell界面。

下面两个步骤非常重要，否则客户端总是提示密码错误！！！

* 首先你必须确保 git-shell 已存在于 /etc/shells 文件中

1. 使用命令which git-shell判断系统是否安装了git-shell。如果已经安装，则返回git-shell的安装目录，如：`/usr/bin/git-shell`；如果未安装则需要安装git-shell命令，安装命令：`yum install git`
2. 判断shells文件是否存在，判断命令：`cat /etc/shells`
3. 如果文件不存在或没有`/usr/bin/git-shell`，则需要使用vim增加这个路径：
   `sudo vim /etc/shells`，在最后一行添加git-shell路径

``` shell
# /etc/shells: valid login shells 
/bin/sh
/bin/dash
/bin/bash
/bin/rbash
/usr/bin/tmux
/usr/bin/screen
/usr/bin/git-shell # 添加你的git-shell
```

* 现在你可以使用 chsh 命令修改任一系统用户的shell权限了
  1. 现在我们修改第一步中创建的git用户的登录权限，禁止git用户使用shell权限：
     终端中输入`sudo chsh git`
  2. 然后在`Login Shell [/bin/bash]`: 后输入git-shell路径`/usr/bin/git-shell`
  3. **修改完成后验证：** `vim /etc/passwd`找到类似`git:x:1000:1000:,,,:/home/git:/usr/bin/git-shell`，看看git用户是否是以git-shell结尾

* 这样，git用户就只能使用SSH连接对Git仓库进行推送和拉取操作，而不能登录机器并取得普通shell命令

### 三、hexo配置

创建hexo的目录。hexo可以理解为用来把md文件生成静态网站的软件。

``` shell
mkdir /data/wzh-blog/hexo
```

然后在该目录下，执行下列命令：

``` shell
cd /data/wzh-blog/hexo
hexo init ./
npm install
```

新建完成后，指定文件夹的目录如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

### 四、nginx配置

最后，为了能让浏览器能直接访问静态页面，需要使用nginx将端口或域名指向hexo静态文件目录。

#### 4.1 先建用来装网站的目录

```shell
mkdir /data/wzh-blog/www
```

#### 4.2 修改 NGINX 的 DEFAULT 设置

```
vi /etc/nginx/nginx.conf
```

**注意**：不同版本的nginx或系统，nginx的配置文件位置不一定相同。我这里是直接修改了nginx.conf文件。也可以直接新建`/etc/nginx/conf.d/blog.conf`来修改，具体看配置文件描述，总之就是修改下nginx的配置。

#### 4.3 将其中的 ROOT 指向装网站的目录

修改root，也就是根目录。可以同时修改域名 server_name为你自己的域名。其他参数不用动。如下：

``` shell
 server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  xxxx.com www.xxxx.com;
        root        /data/wzh-blog/www;
```

#### 4.3 最后重启服务，让NGINX生效

``` shell
service nginx restart
```

#### 4.4 nginx 常见错误

我们在配置完Nginx后，启动后访问网站可能报403错误。可以查看Nginx日志，日志位置看刚才的配置文件`nginx.conf`里面有。或者直接百度Nginx出现403 forbidden的解决方案等等。

我碰到的是权限问题，启动服务的用户和配置文件的用户不一致。直接更改配置文件，把用户改成root：

``` shell
vi /etc/nginx/nginx.conf
#user nginx;
user root;
```



### 五、其他配置

#### 5.1 修改目录及权限

最后还需要把博客相关的git、hexo、www目录都改成git所有，不然钩子脚本可能会执行出错：

``` shell
chown -R git:git /data/wzh-blog/*
chmod -R 755 /data/wzh-blog/*
```

#### 5.2 测试

当上述步骤都完成后，我们就可以测试下git服务器是否部署成功。

最简单的方法便是使用clone来校验：

1. 用户电脑（window or mac）git客户端执行clone操作

   ```shell
   git clone git@服务器ip:/data/wzh-blog/git-repo/blog.git
   ```

2. 然后随便在用户电脑的blog目录里添加一个xxx.md文件。

3. 再提交代码

   ```
   git add .
   git commit -m 'test'
git push
   ```
   

最后看下提交有没有报错，访问网站看下是否更新成功了。

以上就部署完毕了。本文之后的内容，有兴趣可以了解下。

#### 5.3 hexo部署

如果需要把hexo生成的网站文件同时部署到Github等网站，可以如下操作：

修改部署文件

```
vi /data/wzh-blog/hexo/_config.yml
```

在文件最后面，修改repository地址：

```shell
deploy:
  type: git
  repository: git@服务器ip:/data/wzh-blog/git-repo/blog.git 或者github仓库地址
  branch: master
```

然后上述2.4步的脚本中，把`hexo g;` 修改为`hexo g -d`即可。

### 六、主题选择

可以使用原生的主题，也可以使用其他主题。我用的是3-hexo主题。这个主题已经在github上开源，拿来直接用就行。

#### 6.1 拷贝主题

将主题拷贝到Hexo主题目录：

```
cd /data/wzh-blog/git-repo/blog.git;
git clone https://github.com/yelog/hexo-theme-3-hexo.git themes/3-hexo;
mv ./themes/3-hexo /data/wzh-blog/hexo/themes;
rm -f ./themes;
```

修改hexo根目录的`_config.yml`

```
vi /data/wzh-blog/hexo/_config.yml
```

修改them3:

```
theme: 3-hexo
```

然后重新从本地提交代码，触发网站更新即可。

#### 6.2 主题玩法

主要是通过修改`/data/wzh-blog/hexo/themes/3-hexo/_config.yml`文件中的配置来实现。里面讲得很清楚了，文末参考资料也有作者博客的连接。

然后提几个特别点的：

1. 修改首页内容：修改`/layout/indexs.md`即可

2. 更换头像：修改`source/img/avatar.jpg`或是修改`_config.yml` 中头像的配置记录`avatar`即可。

3. 修改赞赏付款码：修改`source/img/alipay.jpg`和`source/img/weixin.jpg`即可

4. 修改关于内容：可以在`_config.yml` 中禁用，也可以在 `hexo` 根目录执行以下，创建 `关于` 页面：

   ```
   hexo new page "about"
   ```

   然后在`source/about/index.md` 根据需要编辑即可。

5. 另外文章最前面需要加上以下参数，控制文章的分类、排序和置顶。top有的话就置顶，参数越大越靠前。

   分类可以建多层，在后面追加就行了，就像下面的-运维-工具。
   
   ```
   ---
   title: 每天一个linux命令
   date: 2017-01-23 11:41:48
   top: 1
   categories:
   - 运维
   - 工具
   tags:
   - linux命令标签
   - 运维标签
   ---
   ```

6. 文章根据时间排序：

   修改`node_modules/hexo-generator-index/lib/generator.js`

   ```
   'use strict';
   var pagination = require('hexo-pagination');
   module.exports = function(locals){
     var config = this.config;
     var posts = locals.posts;
       posts.data = posts.data.sort(function(a, b) {
           if(a.top && b.top) { // 两篇文章top都有定义
               if(a.top == b.top) return b.date - a.date; // 若top值一样则按照文章日期降序排
               else return b.top - a.top; // 否则按照top值降序排
           }
           else if(a.top && !b.top) { // 以下是只有一篇文章top有定义，那么将有top的排在前面（这里用异或操作居然不行233）
               return -1;
           }
           else if(!a.top && b.top) {
               return 1;
           }
           else return b.date - a.date; // 都没定义按照文章日期降序排
       });
     var paginationDir = config.pagination_dir || 'page';
     return pagination('', posts, {
       perPage: config.index_generator.per_page,
       layout: ['index', 'archive'],
       format: paginationDir + '/%d/',
       data: {
         __index: true
       }
     });
   };
   ```

### 6.3 Bug修复

代码行高亮有两个小Bug需要处理。

1.代码块显示的是表格，不是代码

```shell
S# 在hexo主目录，编辑_config_config.yml
highlight:
  enable: false
  line_number: false
```

2.代码块最后一行行号不是空格的话，行号不显示

这个是因为lines计算有误，修改计科

```shell
vim themes\3-hexo\layout_partial\footer.ejs
# 将39行的“-1”去掉
var lines = $(this).text().split('\n').length - 1, widther='';
改为
var lines = $(this).text().split('\n').length, widther='';


```





全文完。



参考资料：

<https://hexo.io/zh-cn/docs/>	作者：hexo官网

<https://www.laoyuyu.me/2017/10/10/hexo_deploy_vps/>	作者：AriaLyy

<https://yelog.org/>	作者：yelog