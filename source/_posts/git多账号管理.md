---
title: git多账号管理
date: 2023-05-05 14:56:42
tags: github, git
categories: git/github
---

为什么提交了代码并推送到github，但是github上的contributions并没有增加呢？

其中Github官方给出了一个官方文件，告诉我们什么样的Commit可以被记入Contribution，请点击此处查看。

在官方的帮助文档中，有一条是Commit被记入Contribution中必须满足用于Commit的邮件地址必须与Github账户相关联。其实，这也是为什么我的Commit没有被记入Contribution和不显示头像的原因，也是大多数人也是这个原因

https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-settings-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile

我遇到的问题的解决方案是：该仓库的本地的邮箱和github账户的邮箱不是同一个配置：可以修改本地仓库的邮箱和github上一致：
git config --local  user.name 张三
git config --local  user.email zhansan@996icu.com

如果拿到一台公司电脑, 那么就请按照下面的最佳实践配置下git的多环境:
请先执行命令打开配置文件
vi ~/.ssh/config

然后输入以下内容：
# gitlab
Host gitlab
    User git
    HostName gitlab.company.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_rsa
    ServerAliveInterval 300
    ServerAliveCountMax 10
# github
Host github
    User git
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/github_rsa
    ServerAliveInterval 300
    ServerAliveCountMax 10

这里唯一需要替换的gitlab里的HostName部分, 改成你们公司的git地址.

遇到GitHub报permission denied错就执行：ssh-add -k ~/.ssh/github_rsa
遇到gitlab报permission denied错就执行：ssh-add -k ~/.ssh/gitlab_rsa


推荐阅读：https://zhuanlan.zhihu.com/p/62071906