---
title: "Git 工作流最佳实践"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Git
categories:
  - 技术
---

记得在学校期间，一直认为Git是个上手门槛很高的工具，但进入工作后才发现其实常用的命令就那么些。其实工作中不在与Git用的有多好，更重要的是心存敬畏，每次推送远程仓库之前仔细检查。

后面主要就两方面来简单说一下常用的Git命令吧，一是正常的开发流程（不出意外）、二是错误处理（出了意外怎么办）。

> 本篇文章只是总结，强烈阅读 Pro Git 的前几章节，加深对Git中阶段和分支的理解。有相应的中文版本，阅读时间大概在1h左右，门槛较低，链接附在文末。

# 日常开发

1. 拉取远程分支

    ```sh
    git clone URL
    git checkout master		 # 切换到master分支
    git checkout -d dev_self # 从master分支的基础上检出自己的开发分支dev_self
    ```

2. 保存代码

    ```shell
    git status					# 查看当前文件的状态
    git add FILE1 FILE2 FILE3   # 将文件FILE1,FILE2,FILE3加入「待提交区」
    git commit "first commit"	# 提交
    git log						# 查看提交记录，确认分支以及提交内容
    ```

3. 推送至远程仓库

    ```shell
    git checkout master
    git fetch origin
    git rebase origin/master
    git log                  # 检查分支是否与远程保持一致
    git pull    			  # 确保本地的master分支是最新的
    git rebase master		# 变基到master分支，如果存在冲突则需要解决冲突，并重新提交
    git push				# 推送到远程仓库
    ```

# 错误处理

1. 我的上次提交的代码存在bug，想撤销删除这次提交怎么办？

    ```shell
    git reset --soft HEAD^1 # 删除最近的一次commit，但保留更改到「待提交区」
    ```

2. 分支命名出现问题，我想改个名字再推送到远程仓库怎么办？

    ```shell
    git branch -m another   # 将现在的分支重命名为another
    ```

3. 我上次提交的代码完全不对，想重新回到起跑线怎么办？

    ```shell
    git reset --soft HEAD^1 # 删除最近的一次commit，并丢弃所有更改
    ```

# 参考

1. [Pro Git](https://www.progit.cn/)
