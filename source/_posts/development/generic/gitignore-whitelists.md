---
title: 白名单模式的 gitignore 配置
date: 2025-09-27 02:34:18
tags:
  - git
  - gitignore
categories:
  - Development
  - Generic
---
WhiteLists

虽然已经用了多年的 git, 但是至今都没怎么好好的总结一下 ignore 规则。今天心血来潮，重新仔细的总结一下 gitignore 到底怎么好好的去写，并记录一下 gitignore 的白名单写法。

# 基础规则

首先是 gitignore 的基础规则。基础规则的总结，我参考了这篇文章 [.gitignore File – How to Ignore Files and Folders in Git](https://www.freecodecamp.org/news/gitignore-file-how-to-ignore-files-and-folders-in-git/). 不过参考文章中有些操作在我所使用的 Archlinux 中无法复现, 所以下面的总结以我自己的实际操作为基础。

## 如何在 Git 中忽略一个文件或文件夹

### 如何忽略一个文件或目录

```
/text.txt           # 忽略根目录下 text.txt 文件
/test/text.txt      # 忽略根目录下 test 目录中的 text.txt 文件

text.txt            # 忽略所有名为 text.txt 的文件和目录
test/               # 忽略所有名为 test 的目录
test                # 忽略所有名为 test 的文件和目录

*test*              # 忽略所有名字带有 test 的文件和目录
img*                # 忽略所有名字开头是 img 的文件和目录
*.md                # 忽略所有名字结尾是 .md 的文件和目录
```

### 如何不忽略一个文件或目录

加入我们忽略了所有md文件, 但唯独不忽略README.md的情况, 如下所示.

```
*.md                # 忽略所有名字结尾是 .md 的文件和目录
!README.md          # 不忽略所有名字是 README.md 的文件和目录
```

### 无法排除已经被忽略的目录内的一个文件

```
test/               # 忽略所有名字带有 test 的目录
!test/example.md    # 试图在被忽略的目录内排除一个文件是行不通的
```

## 如何忽略以前提交的文件

假如你不小心把 .build 文件提交了, 现在想去除这个文件.过程如下所示.

```sh

# 给 .gitignore 添加 .build 文件
echo ".build" >> .gitignore

# 从 git 版本库中去除该文件
git rm --cached .build

# 暂存新的 .gitignore 并提交
git add .gitignore
git commit -m "更新了 .gitignore 文件"

```

# 白名单模式

一般来说 gitignore 是使用黑名单模式，也就是写在 ignore 中的项目会被忽略，但是有些时候使用黑名单模式会有一些疏漏，当项目逐渐庞大起来，难免会有一些遗漏，将一些临时的文件和不希望添加入 git 管理的文件加入 git 中。这个时候白名单模式的 gitignore 能够实现更加精细的管理。

假设我们有以下结构的 C++ 项目需要管理：

```
your-project-dir
├── .gitignore  # gitignore 文件
├── build/      # 构建项目的临时目录
├── docs/       # 放置各种文档
├── include/    # 放置 .h 头文件
├── README.md   # 自述文档
└── src/        # 放置 .cpp 代码文件
```

## **首先忽略全部文件，并取消忽略目录**

白名单模式的第一步。这里取消忽略目录，是因为被忽略目录下的文件是必定被忽略的，为了能够指定不忽略的文件，需要取消忽略目录。

```
*
!*/
```

## **忽略不想要的目录** （可选）

因为 build 目录完全是一个临时目录，我不想提交里面的任何文件，所以可以直接忽略掉。如果这个目录里有某些特定的文件需要提交，则不应该忽略这个目录。

之所以这是一个可选的操作，是因为在上一步忽略了所有文件之后，所有的目录内都没有需要提交的文件，在 git 中这样的“空目录”不会被提交。

```
/build/
```

## **取消忽略特定的文件**

有些文件我们希望必须提交的，我们可以直接取消忽略这些文件。

```
!.gitignore
!README.md
```

这样之后，任何目录下的 .gitignore 和 README.md 文件都会被提交上去。

## **取消忽略特定目录下的全部文件**

我希望提交 docs 目录下的所有文件，包括其子目录下的所有文件。通过下面这种写法，可以取消忽略 docs 目录下的所有文件，并递归的忽略其子目录下的所有文件。

```
!/docs/**/*
```

## **取消忽略特定目录下的特定文件**

有的目录我只希望提交特定类型的文件，例如 src 目录下的 .cpp 代码文件和 include 目录下的 .h 头文件。

下面这种写法取消忽略了对应目录下所有特定后缀的文件，并递归的忽略其子目录下所有特定后缀的文件。

```
!/include/be/**/*.h
!/src/be/**/*.cpp
```

## 总结

最后的白名单 gitignore 文件看上去是这样的：

```
*
!*/

/build/

!.gitignore
!README.md

!/docs/**/*
!/include/be/**/*.h
!/src/be/**/*.cpp
```

在比较复杂的项目中，或者需要精细控制提交的文件时，白名单模式的 ignore 文件是真的很好用。

