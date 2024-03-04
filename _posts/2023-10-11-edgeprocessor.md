---
layout: post
title: EdgeProcessor
date: 2023-10-11 13:59 +0800
tags: [EDA, Klayout, DB]
categories: [Klayout, DB]
author: Jebearssica
---

一个非常通用的扫描线算法流程, 在 Klayout 内部对许多运算中都用到的运算框架. 要注意的是, 由于 Klayout 的项目几乎通过一个人开发, 为了避免代码膨胀减轻维护压力, 所以有大量的代码复用. 然而这是一个双刃剑, 代码维护轻松了, 但是由于一个流程框架涉及众多命令, 因此势必相较于特例化的算法流程而言更加复杂低效.

很难读懂, 一个看似没什么作用的值重置可能都隐含着一类命令在此引发的问题

## XXX
