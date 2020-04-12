### 1.常用提交命令

#### git init

创建版本库。

#### git add .

增加当前文件夹下所有文件到版本管理。

#### git commit -m "xxx"

提交修改到本地仓库。相当于把当前的状态作为快照，保存下来。

#### git push

提交修改到远程仓库。

### 2.代码状态查看x

#### git status

告诉你文件被修改过。查看还没有commit的文件状态。

#### git diff xxx.txt

告诉你文件被修改的内容。查看还没有add的文件，修改的详细信息。xxx.txt也可以不加。

#### git log

查看commit的提交日志。可以加`--pretty=oneline`参数，只显示一行简略信息。

### 3.版本回退

在Git中，用`HEAD`表示当前版本。上个版本是`HEAD^`，上上个版本是`HEAD^^`，上100个版本是`HEAD~100`。

#### git reset

回退到上一个版本：

```shell 
git reset --hard HEAD^
```

这个时候用`git log`查看提交记录，回退前那一版的commit记录已经没有了。

如果我们想再改回原来的最新版怎么搞呢？可以直接用版本号回退。

回退到某个特定版本：

```shell
# 版本号没必要写全，写前几位就行了，git会去自己找。
# 查看所有的修改命令记录可以用git reflog
git reset --hard c8430c2
```

#### git reflog

全部的提交记录，包括reset记录，可以通过`git reflog`查看。

