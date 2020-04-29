---
layout: post
title: git常用命令总结——覆盖日常开发全操作
category: Git
tags: [Git]
---


### 前言：Git是目前世界上最先进的分布式版本控制系统，对的，最先进！

#### 1. 版本库，又名仓库，repository

可理解成一个目录，目录里的所有文件都可被Git管理，Git可以跟踪每个文件的修改、删除等。版本库里最重要的是称为stage（也叫index）的暂存区，然后是Git自动创建的第一个分支master，以及指向master的一个指针叫HEAD。
注意：工作区指电脑上看到的目录，和版本库是不同的概念，工作区的.git文件，是Git的版本库。

#### 2. git init

初始化，把当前目录变成git可以管理的版本库，会生成一个.git文件

#### 3. git add filename

把文件添加到仓库，此时是将修改添加到暂存区

#### 4. git commit -m "本次提交的注释"

把文件提交到仓库，此时是将暂存区的所有内容提交到当前分支

#### 5. git status

查看当前仓库的状态

#### 6. git diff filename

比较文件修改前后的差异

#### 7. git log

显示从最近到最远的提交日志

#### 8. git log --pretty=oneline

显示提交日志，简洁版，不附带过多信息

#### 9. git reset 

版本回退，将当前版本回退到历史中的某个版本
<br/>
用法一：git reset --hard HEAD^ 回退到上个版本，HEAD表示当前版本
<br/>
用法二：git reset --hard HEAD^^ 回退到上上个版本，如果回退到之前100个版本，可以写成HEAD~100
<br/>
用法三：git reset --hard commit_id  回退到commit_id对应的版本号（commit_id表示版本号） 

#### 10. git reflog

查看历史命令，可从显示的命令中找到版本号

#### 11. git diff HEAD -- filename

查看filename文件在工作区和版本库里最新版本的区别

#### 12. git checkout -- filename

撤销filename文件在工作区的修改

#### 13. git rm filename

从版本库删除filename文件

#### 14. ssh-keygen -t rsa -C "youremail@example.com"

创建SSH Key，生成id_rsa私钥和id_rsa.pub公钥

#### 15. git remote add origin 仓库地址

将本地仓库和远程仓库关联起来

#### 16. git push -u origin master

把master分支推送到远程，origin是远程库的名字，这个是Git默认约定的叫法。
注意：第一次加上了-u参数，Git会把本地的master分支内容推送到远程新的master分支，还会把本地master和远程master关联起来，后面就可以去掉-u参数了。

#### 17. git clone 仓库地址

将远程仓库克隆到本地库

#### 18. git checkout -b 分支名称

创建分支，并且切换到该分支
相当于两条命令： git branch 分支名称   git checkout 分支名称

#### 19. git switch -c dev

创建并且切换到新的dev分支

#### 20. git branch

查看分支，列出左右分支，并且在当前分支前面加上*号

#### 21. git merge dev

合并分支，将dev分支合并到当前分支

#### 22. git merge --no-ff -m "merge注释" dev

合并分支时，加上--no-ff参数表示用普通模式合并，合并后的历史有分支，可以通过git log来查看。如果用fast forward合并将看不出曾做过合并。

#### 23. git branch -d dev

删除dev分支， -d改为-D的话，表示强行删除

#### 24. git pull

拉取远程内容

#### 25. git log --graph

查看分支合并图

#### 26. git stash

贮藏当前工作区

#### 27. git stash pop

恢复贮藏的工作区，并把stash内容删除掉

#### 28. git rebase

将分叉的分支重新合并

#### 29. git tag tagName

给当前分支打上标签，默认打在该分支最新提交的commit上

#### 30. git tag

查看所有标签，结果按照字母排序

#### 31. git tag -a tagName -m "标签注释"

指定标签信息

#### 32. git tag -d tagName

删除一个本地标签

#### 33. git push origin tagName

推送指定的标签到远程

#### 34. git push origin :refs/tags/tagName

删除一个远程标签