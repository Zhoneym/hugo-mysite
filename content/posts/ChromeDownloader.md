---
title: "ChromeDownloader 2.0"
tags: ["Google Chrome"]
date: 2023-12-10T19:18:25+08:00
toc:
  depth_from: 1
  depth_to: 4
  ordered: false
draft: false
---


ChromeDownloader 2.0 是一份基于 Java 的 获取Chrome 浏览器离线包的开源 Java 控制台应用程序源码

编译后运行就可以快速以获取最新版本 Google Chrome 不同更新频道的浏览器离线包下载地址和软件包

有人评论或私信反馈 某某网站也提供了下载Chrome 浏览器的地方，好像更新频率还是慢了


与 ChromeDownloader 初始版本只是一个 powershell 脚本，只工作在 Windows 上不同

ChromeDownloader 2.0 支持 Windows 和 MacOS 平台，修复了初始版本所有已知问题，下载软件包的速度变快了

https://github.com/Zhoneym/ChromeDownloader

同时提供可以随时获取多个平台 Chromium 浏览器的PowerShell脚本


# 快速食用指南

安装 bellsoft LibericaJDK：

https://bell-sw.com/pages/downloads/

https://download.bell-sw.com/java/21.0.1+12/bellsoft-jdk21.0.1+12-windows-amd64-full.msi

https://download.bell-sw.com/java/21.0.1+12/bellsoft-jdk21.0.1+12-macos-aarch64-full.dmg

https://download.bell-sw.com/java/21.0.1+12/bellsoft-jdk21.0.1+12-macos-amd64-full.dmg

# 编译

如果你需要 Windows 版本的 Google Chrome ：javac ChromeDownloader.java

如果你需要 MacOS 版本的 Google Chrome ：javac ChromeDownloaderMacOS.java


# 运行


如果你需要 Windows 版本的 Google Chrome ：java ChromeDownloader


如果你需要 MacOS 版本的 Google Chrome ：java ChromeDownloaderMacOS


Select a version with an update frequency that suits you.



# 便携化

Chrome++ : https://github.com/Bush2021/chrome_plus
