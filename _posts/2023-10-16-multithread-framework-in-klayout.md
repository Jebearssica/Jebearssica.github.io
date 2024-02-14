---
layout: post
title: Multithread Framework in Klayout
date: 2023-10-16 17:35 +0800
tags: [EDA, Klayout, Multithread, DB]
categories: [Klayout, DB]
author: Jebearssica
---

Klayout 提供了一套多线程的框架, 在 EDA 这样的计算密集型场景下, 每个核心只需一个线程即可达到高负载. 因此此处的 multithread 可以等价于 multiworker.

## `JobBase`


