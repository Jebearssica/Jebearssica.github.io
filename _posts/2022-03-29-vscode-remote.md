---
layout: post
title: Vscode Remote
date: 2022-03-29 16:19 +0800
tags: [config, bug fix, vscode]
categories: [Linux, Config]
author: Jebearssica
---

本篇博客有关于Linux运用, 涉及Windows-Linux SSH配置 & vscode-remote配置 & vscode-sftp实现本地云端同步

## Windows-Linux SSH配置

参考地址: <https://zhuanlan.zhihu.com/p/102866267>, 其实照着这篇知乎专栏有眼有手就能配

### Requirement

* 一台个人主机(反正我是Windows, 不懂其他的系统不过操作应该差不多)
* 一个Linux云(反正学校给的是啥就是啥, 你也没得挑, 有就不错了)

> **一个能够连上学校服务器的网络**: 非常重要, 外网无法访问学校服务器, 只能连学校内网访问, 不确定后面会不会利用VPN或内网穿透实现外网访问学校内网服务器
{: .prompt-warning }

### 精简后的配置步骤(有问题错了再改)

* SSH key: 检查`C:\Users\\[你的用户名\]\.ssh\`{: .filepath} 有没有这个文件夹, 通常安装了git的电脑都配好了
  * 如果不存在id_rsa与id_rsa.pub就需要进行配置: `ssh-keygen -t rsa -b 4096`

## vscode-remote配置

## vscode-sftp实现本地云端同步

## 关于一些还未解决的疑问

- [x] 我第一次配vscode和linux连接的时候, 我得先用Xshell连上远端, 然后配密钥, 为啥配好了之后用其他人的电脑能够直接连

答: 使用authorized_keys并存入自己的公钥只是能够实现免密登录

## 关于一些Error

> ssh-add: Could not open a connection to your authentication agent
{: .prompt-danger}

添加git rsa私钥到本地时报错, 问题描述为: 无法打开一个身份代理连接

通过下述指令启动一个子进程, 然后再执行私钥添加

```sh
ssh-agent bash
ssh-add ~/.ssh/id_rsa
```

> Vscode can not connect to remote server
{: .prompt-danger}

报错类型

```sh
Failed to parse remote port from server output
```

与Windows自带的openssh-client服务有关, 与.ssh/knowhosts无法修改有关

<https://github.com/microsoft/vscode-remote-release/issues/1123>
