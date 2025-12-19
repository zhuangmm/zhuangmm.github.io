---
title: 如何搭建一个博客
published: 2025-12-19
description: 终于把个人博客搭建起来了，这不得写个教程。
tags: [blog, astro]
category: tutorial
draft: false
---
# 前言
很久之前，我就想搭建一个自己的个人博客了。之前也尝试过几次，但是都不了了之。。。。。。（ps：万事开头难）

最近总算是有时间给他搞起来了，总算是把这个坑给他填上了，ohhhhh。

搞之前，其实还是有点担心的，毕竟我的前端知识虽然不能称为一点没有，但也只能说是聊胜于无。结果实际搞起来，发现还挺顺利的😝。

# 原理
作为一个程序员应该对 `Markdown` 文档不陌生，所以博客项目其实可以理解为将 `Markdown` 转换成一个 `HTML` 静态页面。

知道这个之后，也就明白，如果你对博客的内容进行了修改，就需要重新进行编译、部署。

要完成这个转换工作，我们需要使用一些静态站点生成器了，比如 `Jekyll`、 `Hexo`、 `Astro` 等等。（我选了`Astro`，主要别的没找到喜欢的主题）

主题是个啥呢，主题主要是负责前端的显示效果，毕竟光秃秃的文字做一个页面也不好看，hhhhh。（所以一定要选个自己喜欢的）

# 步骤
知道了原理，我们接下来总结一下搭建博客的步骤
1. 选择一个喜欢的主题和静态站点生成器
2. 修改自定义配置
3. 搭建自动化部署（你也不想每次写个新文章都需要手动编译、部署吧）

## 步骤1 安装静态站点生成器并下载主题
由于 `Astro` 这个静态站点生成器是基于 `Node.js` 的，所以我们需要先安装 `Node.js`，这里是直接走的[官方链接](https://nodejs.org/en/download)安装的，这里选择的版本是 `24`，包管理器选择了 `npm`。

主题的话，选择的是 [fuwari](https://github.com/saicaca/fuwari)，`Astro` 的官网以及 `GitHub` 上面也有一些其他很不错的主题，可以自行选择。
## 步骤2 修改自定义配置
fuwari 这个主题的配置，主要在 `src/config.ts` 里面，可以在其中修改头像、favicon、banner等等。

关于页面位于 `src/content/spec/about.md`，可以自行修改

博客文档位于 `src/content/posts` 目录下

## 步骤3 自动化部署
这里是通过 GitHub Action 部署在了 GitHub Pages上面，参考了 [Astro 的官方部署文档](https://docs.astro.build/zh-cn/guides/deploy/github/)，由于这个文档写的很详细，这里就不再赘述了，直接参考就行。
