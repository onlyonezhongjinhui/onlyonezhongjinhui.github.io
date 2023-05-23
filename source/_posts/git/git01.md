---
title: Git如何剔除某些提交
date: 2021-05-19 11:20:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary:
tags: Git
categories:
- [Git]
---

假如小明同学在某次需求开发中在Git仓库的main分支中分别提交了5次，分别是A->B->C->D->E。几天后突然觉悟了，发现某些功能搞错了，需要把B这个提交的代码剔除。这时候小明就着急了，该如何是好呢？正在这时，小明想起来貌似听说过一个很厉害的命令cherry-pick，这时小明开始变得淡定起来，嘴角上扬，胸有成足。那么它是怎么操作的呢？一起来还原现场吧。

1. git reset A --hard 还原到A这个版本
2. git cherry-pick C D E 把C、D、E cherry pick过来
3. git push --force origin 强制推送到远端仓库

就这样，小明就把B的提交给剔除了。这都多亏了cherry-pick命令了。

## cherry-pick
### 应用于其他分支
```bash
git cherry-pick <commitHash>
```
上面命令就会将指定的提交commitHash，应用于当前分支。这会在当前分支产生一个新的提交，当然它们的哈希值会不一样。

举个栗子，代码仓库有main和dev两个分支。
    a - b - c main
         \
           d - e - f - g dev
要将提交f应用到main分支中去
```bash
git checkout main
git cherry-pick f
```
操作完成以后，代码库就变成了下面的样子。
```
a - b - c - f   main
     \
       d - e - f - g dev
```
从上面可以看到，main分支的末尾增加了一个提交f。

git cherry-pick命令的参数，不一定是提交的哈希值，分支名也是可以的，表示转移该分支的最新提交。
```bash
git cherry-pick dev
```
操作把dev分支的最新一次提交应用到了main分支上，变成下面这样子。
```
a - b - c - g   main
     \
       d - e - f - g dev
```
如果一次性需要应用多个提交，直接命令后填写多个提交即可
```bash
git cherry-pick <commitHash1> <commitHash2> <commitHash3>
```
如果需要应用的提交很多，比如需要应用commitHash1到commitHashN的所有提交，可以这样操作
```bash
git cherry-pick <commitHash1>..<commitHashN>
```
这样，就可以把从commitHash1到commitHashN的提交全应用上去了。不过需要注意的是，这是不包含commitHash1的，且commitHash1到commitHashN的顺序不能颠倒，否则会失败但不报错。如果需要包含commitHash1，则只需要稍加修改
```bash
git cherry-pick <commitHash1>^..<commitHashN>
```
### 转移到另一个仓库
```bash
git remote add target git://gitUrl
```
添加远程分支
```bash
git fetch target
```
将远程分支拉取到本地
```bash
git log target/main
```
从日志中获取提交的hash值
```bash
git cherry-pick <commitHash>
```
应用提交即可
### cherry-pick常用配置项
1. -e，--edit
打开外部编辑器，编辑提交信息
2. -n，--no-commit
只更新工作区和暂存区，不产生新的提交
3. -x
在提交信息的末尾追加一行(cherry picked from commit ...)，方便以后查到这个提交是如何产生的
4. -s，--signoff
在提交信息的末尾追加一行操作者的签名，表示是谁进行了这个操作
5. -m parent-number，--mainline parent-number
如果原始提交是一个合并节点，来自于两个分支的合并，那么 Cherry pick 默认将失败，因为它不知道应该采用哪个分支的代码变动。
-m配置项告诉 Git，应该采用哪个分支的变动。它的参数parent-number是一个从1开始的整数，代表原始提交的父分支编号。
### 代码冲突策略
如果操作过程中发生代码冲突，Cherry pick 会停下来，让用户决定如何继续操作
1. --continue
用户解决代码冲突后，第一步将修改的文件重新加入暂存区（git add .），第二步使用下面的命令，让 Cherry pick 过程继续执行。
```bash
git cherry-pick --continue
```
2. --abort
发生代码冲突后，放弃合并，回到操作前的样子
3. --quit
发生代码冲突后，退出 Cherry pick，但是不回到操作前的样子
