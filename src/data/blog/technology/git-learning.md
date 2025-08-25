---
author: 迷途的汪汪
pubDatetime: 2025-07-14T12:00:00Z
modDatetime: 2025-07-15T12:00:00Z
title: git学习记录
slug: git_learning_record
featured: false
draft: false
tags:
  - Git
description: git常用命令记录
---

> 课程链接: [Git 和 GitHub 视频教程](https://www.bilibili.com/video/BV1124y117Dr/?spm_id_from=333.1387.upload.video_card.click&vd_source=294ddfe7b7abecfae5ed497431cddc14)

## 1. 初始化仓库

#### 配置

```bash
git config --global user.name "Your Name"
git config --global user.email "Your email"
```

#### 使用 git：

`git status`: 查看当前仓库的状态。

`git init`: 初始化一个新的 Git 仓库。

## 2. 文件状态:

`git`中文件有两种状态：未跟踪和已跟踪。

未跟踪是指文件没有被`git`所管理。

已跟踪是指文件已被`git`管理。已跟踪的文件有三种状态：未修改、修改和暂存。
暂存：表示文件修改已保存，但是尚未提交到 git 仓库。
未修改：表示磁盘中的文件和 git 仓库中文件相同，没有修改
已修改：表示磁盘中文件已被修改，和 git 仓库中文件相同。

可以通过`git status`查看文件状态。

#### 状态转换

##### 未跟踪-->暂存。

`git add <file>`：将文件切换到暂存的状态，刚刚添加到项目中的文件处于未跟踪状态。

`git add *`: 将所有已修改(未跟踪)的文件暂存。

##### 暂存-->未修改。

`git commit -m "xxxx"`: 将暂存的文件存储到仓库中。

`git commit -a -m "xxxx"`: 提交所有已修改的文件(未跟踪的文件不会提交)

##### 未修改-->修改

修改代码后，文件会变为修改状态。

## 3. 常用命令

#### 重置文件

```bash
git restore <filename> # 重置文件到上次提交的状态

git restore --staged <filename> # 取消暂存状态

```

#### 删除文件

```bash
git rm <filename> # 删除文件
git rm <filename> -f # 强制删除文件
```

#### 移动文件/重命名文件

`git mv <filename> <new_filename>`

## 4. 分支

1. `git`在存储文件时，每一次代码的提交都会创建一个与之对应的节点，git 就是通过一个一个的节点来记录代码的状态的。
2. 节点会构成一个树状结构。树状结构就意味着这个树会存在分支，命名为`master`。
3. 在使用`git`时，可以创建多个分支，分支与分支之间相互独立，在一个分支上修改代码不会影响其他的分支。

```bash
git branch # 查看当前分支
git branch <branch_name> # 创建分支
git branch -d <branch_name> # 删除分支
git switch <branch_name> # 切换分支
git switch -c <branch_name> # 创建并切换分支
```

在开发中，都是在自己的分支上编写代码，代码编写完成后，再将自己的分支合并到主分支中。

####

在开发中，除了使用 merge 来合并分支外，还可以使用 rebase 来合并分支。

我们通过`merge`合并分支时，在提交记录中会将所有的分支创建和分支合并的过程全部都现实出来，这样当项目比较复杂，开发过程比较波折时，必须要反复的创建、合并、删除分支。这样一来会使得我们代码的提交记录变得极为混乱。

原理(变基时发生了什么):

1. 当我们发生`rebase`时，git 会首先找到两条分支的最近的共同祖先。
2. 对比当前分支相对于祖先的历史提交，并且将它们存储到一个临时文件中。
3. 将当前部分指向目标的基底
4. 以当前基底开始，重新执行历史操作。

变基和 merge 对于合并分支来说最终的结果是一个样的！但是变基会使得代码的提交记录更整洁更清晰。

**注意**: 大部分情况下合并和 rebase 是可以互换的，但是如果分支已经提交给了远程仓库，那么这时尽量不要变基

## 5. 远程仓库

目前对于 `git` 所有操作都是在本地进行的。在开发中显然不能这样，这时我们需要一个远程的 `git` 仓库。远程的 git 仓库和本地的本质没有什么区别。不同点在于远程的仓库可以被多人同时访问使用，方便我们协同开发。在实际工作中，`git`的服务器通常由公司内部使用或是购买一些公共的私有`git`服务器。

```bash
# git remote add origin <url>
git remote add origin https://github.com/lostwangwang/git-demo.git

# git branch：这是 Git 中用于管理分支的基础命令。
# -M：这是一个强制重命名选项，即使新分支名已存在，也会覆盖它。如果使用 -m（小写），则只有在新分支名不存在时才能重命名。
# main：这是指定的新分支名称。在这个命令中，当前所在的分支将被重命名为 main。
git branch -M main

# git push：这是 Git 中用于将本地仓库的提交推送到远程仓库的基础命令。
# -u：这是 --set-upstream 的简写选项。它的作用是在推送的同时，将本地分支与远程分支建立关联。这样在后续操作中，你可以直接使用 git push 或 git pull 命令，而无需每次都指定远程仓库和分支名。
# origin：这是远程仓库的默认名称。当你克隆一个仓库时，Git 会自动将源仓库命名为 origin。
# main：这是要推送的本地分支名称，同时也是远程仓库中对应的分支名称。在这个命令中，本地的 main 分支将被推送到远程仓库的 main 分支。
git push -u origin main
```

#### 远程库的命令

```bash
git remote # 列出当前的关联的远程库

git remote add <远程库名> <远程库地址> # 添加远程库

git remote remove <远程库名> # 删除远程库

git push -u origin main # 向远程仓库推送代码，并和当前分支关联。

git push <远程库> <本地分支>:<远程分支>

git clone <远程库地址> # 克隆远程仓库到本地

git push # 如果本地库的版本低于远程库，push 默认是推不上去的
git fetch # 要想推送成功，必须先确保本地库与远程库的版本一致，fetch它会从远程仓库下载所有代码，但是它不会将代码和当前分支自动合并。
# 使用fetch拉取代码后，必须要手动对代码进行合并。

git pull # 从服务器上拉取代码并自动合并
```

> 注意: 推送代码之前，一定要先从远程库中拉去最新的代码

## 6. tags

- 当头指针没有指向某个分支的头部时，这种状态我们成为分离头指针 `HEAD detached` 在分离头指针的状态下，我们也可以操作代码，但是这些操作不会出现在在任何分支上，所以注意不要在分离头指针的状态下进行来操作仓库。
- 如果非得要回到后边的节点对代码进行操作，则可以选择创建分支后再操作。

```bash
git switch -c <分支名> <提交id>
```

- 可以为提交记录设置标签，设置标签以后，可以通过标签快速的识别出不同的开发节点。

```bash
git tag
git tag <标签名> # 为当前的提交记录设置标签
git tag <标签名> <提交id> # 为指定的提交记录设置标签

git push 远程仓库 标签名
git push 远程仓库 --tags # 推送所有标签到远程仓库
git tag -d <标签名> # 删除标签
git push 远程仓库 --delete 标签名 # 删除远程标签
```

## 7. gitignore

- 默认情况下，git 会监视项目中所有内容。但是有些内容比如 `node_modules`目录中的内容，我们不希望它被`git`所管理。我们可以在项目目录中添加一个 `.gitignore`文件，来设置那些需要`git`忽略的内容。

## 8. gh-pages

- 在 github 中，可以将自己的静态页面部署到 github 中，它会给我们提供一个地址使得我们的页面变成一个真正的网站，可以供用户访问。
- 要求：
  - 要求静态页面的分支必须是 `gh-pages`
  - 如果希望页面可以通过 xxx.github.io,则需要将库名修改为 `xxx.github.io`

## 9. Docusaurus

> [Docusaurus](https://docusaurus.io/docs)

`docusaurus.config.ts`: 配置文件
