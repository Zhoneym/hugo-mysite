---
title: "关于 Windows 10/11 上的 Xbox Live 网络连通性问题的解决方案" 
date: 2024-01-16T11:23:25+08:00
tags: ["Windows"]
categories: ["Windows"]
toc:
  depth_from: 1
  depth_to: 4
  ordered: false
draft: false
---

Windows 进入 Windows 11 开发周期之后微软放弃了旧版 Xbox 应用，Xbox Live 网络连通性一直是一个难解的问题，受影响的游戏有

- 极限竞速 - 地平线 4
- 极限竞速 - 地平线 5
- Grand Theft Auto Online

这些游戏在 Windows 11 上经常会遇到不能联机或者联机网络质量很差的问题，最近 2023-10 Windows 10 22H2 服务堆栈更新进一步删除了 Xbox Live 网络连通性检测功能，本文将提供一个可靠的解决方案。

# 调节运营商网络设备

家用光纤入户宽带接入设备是一种 GOPN 智能网关，安装宽带的时候运营商并不提供管理员账户和密码，只提供了一个用户账户和密码

## 获取管理员账户的登录密码

- 中国联通：这个是固定的 CUAdmin
- 中国电信：密码随机，需要去抓旧版本电信宽带 APP 的 HTTP/HTTPS 请求构造一个查询管理员密码的 HTTP/HTTPS 请求报文

```json
{ 
    "Params": [], 
    "MethodName": "GetTAPasswd", 
    "RPCMethod": "CallMethod", 
    "ObjectPath": "/com/ctc/igd1/Telecom/System", 
    "InterfaceName": "com.ctc.igd1.SysCmd", 
    "ServiceName": "com.ctc.igd1"
}
```

## 调整 NAT 特殊行为

你需要做的是 启用通用即插即用协议（英语：Universal Plug and Play，简称UPnP）并调节防火墙等级为 L3

在路由器上配置 mesh 组网或者 DHCP 中继，由 GPON 负责 PPPoE 拨号

# Windows 上开启 Teredo 通信隧道

设置 Teredo 客户端状态为企业版客户端，默认限定改为启用并开启相关 Windows 服务

