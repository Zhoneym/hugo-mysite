---
title: "关于 VSCode 上 Go 语言开发环境的设置" 
date: 2024-01-26T11:23:25+08:00
toc:
  depth_from: 1
  depth_to: 4
  ordered: false
draft: false
---

# 关于 VSCode 上 Go 语言开发环境的设置

老生常谈的话题，网络教程一大堆；正所谓天下文章一大抄，抄来抄去有提高。🤣 已经2402年了，抄来抄去没提高


## 问题出在哪

其实最主要的问题是 Visual Studio Code 的 Go 语言扩展组件需要下载一些 Go 语言的工具包才能正常运行。

这些工具包包括 `gopls`，`gotests`，`impl`，`goplay`，`dlv`，`staticcheck` 等等。

这些工具包是通过 Go 的包管理工具 `go get` 或者 `go install` 来下载和安装的。

但是 Google 公司提供的服务是被 GFW 和谐了的政治不正确的互联网服务，这些工具包提供了诸如构建、格式化、自动完成等功能，这些功能是 VSCode 的 Go 语言扩展组件所依赖的。

当你安装 Go 语言扩展组件后，VSCode 会提示你安装这些工具包作为开发环境所必须。

所以全网人云亦云的就去给 Go 环境变量加代理。环境变量一顿瞎配 🤣。配完还是不行浪费时间。

## CSDN 迷惑行为大赏

具体的就是你用中文互联网搜索引擎搜索 - 关于 VSCode 上 Go 语言开发环境的设置

你就按 CSDN 大佬的方法去做，你还是搭不起来环境，但是大佬写出了完美结论。

Windows 还是那个 Windows ，Linux 还是那只企鹅，唯独 Go 语言变成了编译器搞了好几天的玄学。

36除以6，除了6还是6

## 正确做法

是 Visual Studio Code 的 Go 语言扩展组件需要下载一些 Go 语言的工具包才能正常运行。所以是 VSCode 在调用下载进程

你只需要在 Visual Studio Code 设置里填入合适的 HTTP 代理服务器这个问题就解决了 😂😂😂