---
title: "Git Tutorial"
date: 2020-06-12T08:24:45+08:00
draft: false
---


创建并切换分支

> git checkout -b name

设置本地分支追踪远程分支

> git push --set-upstream origin `refs/for/name`

备份当前工作区的内容，保存到git 栈中，从最近的一次commit中读取相关内容

> git stash 

- git branch --set-upstream-to=origin/`refs/for/name`
- git pull --rebase

将两个分支融合成一个线性的提交，不会形成新的节点

> git pull --rebase origin `refs/for/name`

从git栈中获取到最近一次stash进去的内容，恢复工作区的内容。获取之后，会删除栈中对应的stash。查看所有的stash (git stash list)

> git stash pop

> git add .

> git commit -m "Lava-#issue: **Add**/**Remove**/**Fix**/**Rework**/**Update**/**Refactor**/**Release**/**Merge**"

既可以对上次提交的内容进行修改，也可以修改提交说明。

> git commit --amend

> git push --force origin name:`refs/for/name`

```tex
Fix failing CompositePropertySourceTests
Rework @PropertySource early parsing logic
Add tests for ImportSelector meta-data
Update docbook dependency and generate epub
Refactor subsystem X for readability
Update getting started documentation
Remove deprecated methods
Release version 1.0.0
Merge pull request #123 from user/branch
```

捡别的分支的 Commit 过来合并
> git cherry-pick 6a498ec

rebase 可以尽可能保持 master 分支干净整洁，并且易于识别 author

- 先切换到 devel 分支（不一样咯）：git checkout devel
- 变基：git rebase -i master
- 切换回目标分支：git checkout master
- 合并: git merge devel

squash 也可以保持 master 分支干净，但是 master 中 author 都是 maintainer，而不是原 owner

- 切换到目标分支：git checkout master
- 以 squash 的形式 merge：git merge --squash devel

merge 不能保持 master 分支干净，但是保持了所有的 commit history，大多数情况下都是不好的，个别情况挺好

- 先切换到目标分支（master）
- 执行命令：git merge devel
- 删除旧分支（可以在上面一同做）：git branch -D devel
- 提交到远程分支：git push origin master

> git config --global user.email "48374308+HenrikZhu@users.noreply.github.com"

编辑提交消息，但在日志中保留了以前的电子邮件地址

> git rebase -i
> git rebase -i HEAD~[X]

替换有问题的电子邮件地址

> git commit --amend --reset-author

> git rebase --continue

> git push

将当前的HEAD重置为指定状态

- --mixed 保留工作目录，并清空暂存区
- --soft 保留工作目录，并把重置 HEAD 所带来的新的差异放进暂存区
- --hard 重置stage区和工作目录

> git reset (--mixed) HEAD^

[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](https://picx.xpoet.cn/)

