---
title: Linux命令速查
date: 2019-08-01 02:22:26
categories:
- 运维
tags:
- Linux
top:
---

[TOC]

### 〇、基础命令

#### 1.命令: date, cal, bc
linux区分大小写。
选项中的"--"表示完整选项名。

```
# 显示时间
date 
```
```
# 显示日历
cal
```
```
# 使用计算器
bc
```
#### 2.ctrl+c, ctrl+d, tab
ctrl+c: 中断当前程序
ctrl+d: 相当于exit
tab: 自动补全

#### 3.man, info
```
# 查看帮助文件
man vim
```

#### 4.关机: who, ps, netstat, shutdown, reboot, halt, poweroff

```
# 数据写入磁盘
sync
```

```
# 关机
shutdown [-t 秒] [-rkhncfF][警告信息]

选项与参数：
-t secNum: 过n秒后关机
-k: 不关机，只是发警告信息
-r: 将系统服务停掉后重启(常用)
-h: 将系统服务停掉后关机(常用)
-n: 不经过init程序，直接用shutdown关机
-f: 关机并开机之后，强制略过fsck的磁盘检查
-F: 系统重启后，强制进行fsck的磁盘检查
-c: 取消已经在进行的shutdown指令内容

范例：
shutdown -h 10 'Linux will shutdown after 10 mins'

-h参数：
now: 现在
20:23: 离现在最近的固定时间点
+10: 十分钟后
```

```
# 重启
# 和上面的shutdown类似

reboot
范例：sync; sync; sync; reboot

halt

poweroff
```

#### 5.切换执行等级: init

```
init 0

参数：
0: 关机
3: 纯文本模式
5: 含有图形接口模式
6: 重启
```
如果忘记开机密码，可以用类似windows的单用户模式，进入系统后修改。

#### 6.查看配置信息

##### 物理cpu数(即插槽)，核数，逻辑cpu数：

CPU总核数 = 物理CPU个数 * 每颗物理CPU的核数 
总逻辑CPU数 = 物理CPU个数 * 每颗物理CPU的核数 * 超线程数

```shell
查看CPU信息（型号）
[root@AAA ~]# cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
     24         Intel(R) Xeon(R) CPU E5-2630 0 @ 2.30GHz

# 查看物理CPU个数
[root@AAA ~]# cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
2

# 查看每个物理CPU中core的个数(即核数)
[root@AAA ~]# cat /proc/cpuinfo| grep "cpu cores"| uniq
cpu cores    : 6

# 查看逻辑CPU的个数
[root@AAA ~]# cat /proc/cpuinfo| grep "processor"| wc -l
24
```

https://www.cnblogs.com/bugutian/p/6138880.html

https://www.cnblogs.com/MC1225/p/8109354.html

https://www.cnblogs.com/xd502djj/p/4318289.html


### 一、文档权限

#### 1.文件属性

````
drwxr-xr-x 6 www www  4096 Jul  4  2018 log
````

档案类型：
d: 目录，directory
-: 档案，
l: 快捷方式，link
b: 区块设备档，如磁盘，block
c: 字符设备档，串行的，如键盘鼠标等，character
s: 数据接口文件，sockets
p: 数据输送管道，主要解决多个程序同时存取一个档案造成的错误问题。pipe

rwx: 拥有者权限，读、写、执行
r-x: 群组权限
r-x: 其他人权限

6: 连接数，有多少不同的档名连结到相同的一个 i-node，后续再讲。

www: 所属用户
www: 所属群组

4096: 文件大小，单位默认bytes

Jul 4 2018: 文档最后修改时间

log: 文档名称，单一最长255，包含路径最长4096

#### 2.文档权限修改: chgrp, chown, chmod




```
# 修改所属用户组
chgrp 用户组 文档名

参数：
-R 递归修改所有子目录及文件

范例：
chgrp usersgroup install.log
```



```
# 修改所属用户
chown 用户名 文档名

参数：
-R 递归修改所有子目录及文件

范例：
chown userowner install.log

其他用法：
同时修改组和用户：加 : 或.
chown user:group install.log

只修改组：加.
chown .group install.log
```


```
# 修改权限
chmod [-option] 权限代码 文档名

参数：
-R: 递归修改所有子目录及文件

r: 4
w: 2
x: 1

范例：
chmod -R 640 install.log

其他用法：
u/g/o/a user，group，other，all
+/-/= 增加，删除，设定
r/w/x 读，写，执行

范例：
chmod u+r install.log
```

### 二、文档操作

#### 1.目录路径操作: cd, pwd, mkdir, rmdir

```
# 切换目录
cd [相对或绝对路径]

参数：
. :   当前目录
~ : 当前用户主文件夹
- : 前一个工作目录
~ : user user用户的主文件夹
```

```
# 显示当前路径
pwd [-P]

参数：
-P 如果是在超链接路径里，则加这个参数显示的是实际路径，而不是链接路径
```

```
# 新建目录
mkdir
```

```
# 删除目录
rmdir
```

ls, cp, rm, mv
cat, tac, nl, more, less, head, tail, od, touch
umask, chattr, lsattr, SUID, SGID, SBIT,file
which, whereis, locate, find

### 三、磁盘操作

dd, df...

### 四、压缩打包备份

compress, gzip, zcat, bzip2, bzcat
tar
dump, restore
mkisofs, cdrecord
dd, cpio

### 五、vim的使用

### 六、bash这个shell

### 七、正则表达式

### 八、shell脚本

### 九、帐号管理

### 十、磁盘分区及磁盘阵列

### 十一、定期自动运行脚本

### 十二、进程管理

### 十三、后台进程与服务

### 十四、登录/启动日志

### 十五、开机加载流程及原理

### 十六、系统设定：防火墙打印机等

### 十七、源码软件编译及安装

### 十八、外部软件安装

### 十九、yum命令

https://www.runoob.com/linux/linux-yum.html

#### 1.语法

### 十九、图形界面x-window

### 二十、数据备份

### 二十一、Linux内核编译与安装