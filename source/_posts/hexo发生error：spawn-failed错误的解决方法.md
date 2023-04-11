---
title: hexo发生error：spawn failed错误的解决方法
date: 2023-04-11 20:59:46
categories: 踩坑
tags: hexo
description: |
    **Spawn failed...**
---

# 问题描述：

先是出现错误：

```
error：spawn failed...
```

然后经过一些博客的操作会出现以下问题：

```
fatal: cannot lock ref 'HEAD': unable to resolve reference HEAD: Invalid argument error: src refspec
```

或者：

```
error: src refspec HEAD does not match any.
```

![](1.png)



问题原因：问题大多是因为`git `进行`push`或者`hexo d`的时候改变了一些`.deploy_git`文件下的内容。

# 解决办法：

- 删除`.deploy_git`文件夹;
- 输入`git config --global core.autocrlf false`
- 然后，依次执行：
  `hexo clean`
  `hexo g`
  `hexo d`

问题解决。暴力直接，有效。



来源：[(57条消息) hexo发生error：spawn failed错误的解决方法_error: spawn failed_HuangTLhit的博客-CSDN博客](https://blog.csdn.net/HTL2018/article/details/106876940)

