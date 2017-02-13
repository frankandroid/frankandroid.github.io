---
title: git的常用命令
date: 2016-03-7 17:46:45
updated: 2016-03-7 17:46:45
categories: [git]
tags: [git]
---

###  git add .     添加到版本库管理

###  git commit - m "提交到版本库"    提交到版本库

###  git push     提交到远程

###  git branch    查看分支

###  git branch -vv   查看本地分支及远程分支以及他们之间的关联

###  git clone git@github.com:michaelliao/gitskills.git         从远程仓库clone

###  git checkout dev     切换到dev分支

###  git checkout origin/daily/1.4.1            git 从远程仓库clone指定分支

###  git checkout -b dev    创建dev分支并切换到dev分支

###  git merge dev  把dev分支合并到当前分支

###  git status 查看当前的状态

### git branch -d dev  删除dev分支

### git reset --hard 3628164   将代码切换到某个版本

### git log  可以查看提交历史，以便确定要回退到哪个版本

### git reflog  查看命令历史，以便确定要回到未来的哪个版本。

### git remote add origin git@github.com:michaelliao/learngit.git 与远程仓库建立联系

### git push -u origin master 我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令

### git branch --set-upstream-to=origin/dev  本地分支同远程的dev分支建立关联

### git branch -u origin/dev  本地分支同远程的dev分支建立关联

### git tag v1.0  打标签 v1.0

### git tag   查看所有标签

### git tag -a v0.1 -m "version 0.1 released" 3628164  打标签并附带说明

### git show <tagname> 查看标签说明

###  git tag -d v0.1 删除标签

### git push origin --tags 一次性推送全部本地标签

### git push origin v1.0  推送某个标签到远程

### git tag v0.9 6224937  对commit id 为6224937的提交打标签

### 如何删除远程标签

git tag -d v0.9
git push origin :refs/tags/v0.9





