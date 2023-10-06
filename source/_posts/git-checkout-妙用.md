---
title: git checkout 妙用
date: 2023-05-05 17:13:52
tags: github, git
categories: git/github
---

当你在 Git 中处理一个仓库时，你可能需要从另一个分支中获取（checkout）一个指定的文件。

幸运的是，Git 提供了许多可能的方法来快速完成这项任务。其中一个最简单的解决方案是使用git checkout命令，将指定的文件作为一个参数。

在这篇文章中，我们将分析这个问题的不同解决方案，并介绍每一种解决方案所需要遵循的流程。

让我们开始吧。😎

Git Checkout 用例
你正在处理一个名为feature/A的分支，其中包含一个名为utils.js的文件。

你有另一个名为feature/B的分支，里面有一个更新的utils.js文件。

你想签出该文件，并将其从feature/B分支带到feature/A分支。

下面是这个任务的三种可行的解决方案。

解决方案 1：使用 git checkout 命令
git checkout命令提供了一个简单的方法来从另一个分支获取文件或文件夹。

以下是从另一个分支获得（checkout）文件的语法：

git checkout <other-branch-name> -- path/to/your/folder
以下是要遵循的流程：

1. 切换（checkout）到你想复制文件的那个分支。

git checkout feature/A
2. 你在当前分支上，复制该文件。

git checkout feature/B -- utils.js
3. 使用git status命令来确保文件已经被复制了。

4. 提交并推送到远程。

在使用 checkout 命令时，你也可以得到：

一个来自另一个分支的文件夹。
通过指定每一个文件，可以指定多个文件
另外，请注意，你可以从 stash 获得一个文件/文件夹。

解决方案 2：使用 git restore 命令
另一个选择是使用git switch命令和git restore命令。

如果你从未听说过这两个命令，那也没关系。它们是比较新的。Git 在 2019 年的 2.23 版本中引入了它们。

这两个命令的目的是减少使用 git checkout命令，以简化用户的工作。

git restore命令可以恢复工作树。

git switch命令切换分支。

下面是从另一个分支获取文件的过程：

1. 切换到你想获取（checkout）文件的分支。

git switch feature/A
2. 从另一个分支获取文件。

git restore --source feature/B -- utils.js
3. 提交并推送更改。

解决方案 3：使用 git show 命令
最后，我们可以使用git show命令。

以下是要遵循的流程：

1. 切换到工作分支。

git switch feature/A
2. 从另一个分支获取文件。

git show feature/B:path/utils.js > path/utils.js
3. 提交并推送更改。

注意: 这次你需要指定来自你的目录根的相对路径。

小结
正如你所看到的，从另一个分支获取一个文件并不是什么像发射火箭那样困难。

当我在日常生活中需要这样做时，我通常使用git checkout命令。