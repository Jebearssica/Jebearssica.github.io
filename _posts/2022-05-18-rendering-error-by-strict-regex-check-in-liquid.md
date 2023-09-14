---
layout: post
title: Rendering Error by Strict RegEx Check in Liquid
date: 2022-05-18 14:14 +0800
tags: [config, bug fix]
categories: [Others]
author: Jebearssica
---

属实是离谱啊, 没见过这种报错的. 两个紧邻大括号会直接导致 Liquid 报错

```sh
Liquid Exception: Liquid syntax error (line 4): Variable '{ {0,1}' was not properly terminated with regexp: /\}\}/
```

> 我服了, 上面报错信息都不能直接写上, 得用空格间隔开
{: .prompt-danger }

你要想复现, 可以把这个 `{ {0,1}, {1,0} }` , 首尾的大括号之间的空格删掉.

看了一下 jekyll 和 Liquid 的 issue, 甚至是 Chirpy Theme 的, 都有人提到. 总之锅的源头指向 Liquid 的过于严格的正则表达式解析? 似乎是为了适配 Liquid-C ? 这 issue 我甚至提都不知道怎么提, 先放这里等等看吧.
