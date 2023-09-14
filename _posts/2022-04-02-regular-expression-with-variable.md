---
layout: post
title: Regular Expression with variable
date: 2022-04-02 16:40 +0800
tags: [RegEx]
categories: [Others]
author: Jebearssica
---

用微软出的PowerToy批量重命名, 想给一长串名字精简一下, 具体场景如下

* 原串: YS.S01E01.巴拉巴拉吧.mp4
* 目标串: S01E01.MP4

按理说应该取得中间要的部分当成一个变量, 然后用这个变量重命名. 可我会取不会存, 这里就记录一下

```RegEx
(.*).(.*).(.*).(.*)
$2.$4
```

> 你得用括号括起来, 这就代表了变量
{: .prompt-tip }
