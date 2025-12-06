---
layout: post
title: "远程服务器连接本地代理"
date: 2025-12-06
tags: [科研日记]
toc: true
author: WqW-hit
---

# 远程服务器连接本地代理

今天这份博客主要记录如何让远程服务器通过走本地代理的方式实现对一些境外网站的快速访问（着急的朋友可以直接看Part 2）

## 1 背景

从huggingface上下载数据集或者使用API调用大模型，这些操作在本地上可以说是相对非常简单的。但是，当这些操作被放到远程Linux服务器上时却出现了一个很基本但很难的问题：如何访问这些网站？

当然，在GitHub上也有类似[wnlen/clash-for-linux: clash-for-linux](https://github.com/wnlen/clash-for-linux)的项目帮助你在Linux系统上实现魔法——我自己在遇到第一次遇到这个问题的时候也是这么做的。但当我按照它的readme完成了一切设置，满心欢喜的准备魔法时却发现魔法还是失败了。

我百思不得其解，之后和同学交流这个问题的时候给了我一个很好的角度：**由于我用的是学校的服务器，学校的服务器在安装/配置的过程中可能从硬件层面ban掉了使用clash等工具进行魔法**。所以哪怕我按照clash-for-linux项目的readme完成了所有的配置，甚至提示“代理已开启”也没有正常实现他应该有的魔法功能

所以接下来我们将要使用另外一个更为直接的方法实现远程服务器魔法——直接将服务器连接本地代理

## 2 具体方法

下面以Windows系统为例介绍如何进行。

首先你要打开你自己在本地的妙妙小工具并确保可用

此后打开你的cmd（命令提示行），在页面内输入

```bash
ssh -R 127.0.0.1:<virtual_port>:127.0.0.1:<virtual_port> <username>@<address> -p <port>
```

对这行指令做一些解释：

1. virtual_port的含义为你本地的魔法工具（如clash）中设置的端口，比如说7900/7897等，要根据你的设置做一些处理
2. 后续的port的含义是你的账户连接服务器的端口，一般来说默认端口为22

在输入这一行指令并且输入账户密码之后，在你的clash或者其他类似软件中找到“复制环境变量类型”，在确保环境变量类型为bash的前提下可以复制得到类似下面的指令

```bash
export https_proxy=http://127.0.0.1:<virtual_port>http_proxy=http://127.0.0.1:<virtual_port> all_proxy=socks5://127.0.0.1:<virtual_port>
```

再将上面的指令复制到终端之后理论上配置就已经完成了，可以运行下面的指令进行尝试

```bash
wget google.com
```

```
--2025-12-06 08:11:47--  http://google.com/
Connecting to 127.0.0.1:7900... connected.
Proxy request sent, awaiting response... 301 Moved Permanently
Location: http://www.google.com/ [following]
--2025-12-06 08:11:48--  http://www.google.com/
Reusing existing connection to 127.0.0.1:7900.
Proxy request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘index.html’

index.html                        [  <=>                                             ]  17.33K  72.4KB/s    in 0.2s

2025-12-06 08:11:49 (72.4 KB/s) - ‘index.html’ saved [17744]
```

如果你能够得到类似上面的输出就说明已经成功的连接了，接下来直接处理就好了
