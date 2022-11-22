---
title: "TiDB MVCC GC"
date: 2022-11-22T23:34:28+08:00
lastmod: 2022-11-22T23:34:28+08:00
draft: false
keywords: []
description: ""
tags: []
categories: []
author: ""

# Uncomment to pin article to front page
# weight: 1
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false

# Uncomment to add to the homepage's dropdown menu; weight = order of article
# menu:
#   main:
#     parent: "docs"
#     weight: 1
---

<!--more-->

TiDB 事物基于 MVCC 实现，新的数据写入，旧的数据依旧存在，通过时间戳进行区分，成为数据的不同版本。此处的 GC 指的是对旧数据进行清除。

TiDB 集群中，存在一个节点成为 GC Leader，负责清除老旧的 MVCC 版本数据。GC 会定期触发，默认 10 分钟一次。

GC 的时候，会默认保留最近 10 分钟内的老旧数据，该时间被称为 “GC life time”。当前时间，减去 “GC life time” 得到的时刻，被称为 “GC Safe Point”，即在该时刻之后的数据，都被保留。

GC 可能是一个颇为耗时的过程，如果当前触发的 GC 过程正在执行中，下一次触发 GC 到达，那么新触发的 GC 不会执行。

GC 执行过程中，不应该影响其他正在执行的事物。如果存在执行时间较长的事物，那么 GC Safe Point 不会大于正在执行的事物的 start_ts。也就是说，GC Safe Point 的计算公式为：

```
GC Safe Point = min(current_time - gc_life_time, start-ts of all running transactions)
```

## GC 的执行过程

第一阶段执行 "resolve lock" 操作，清除所有 regions 上 safe point 之前的锁。

第二阶段执行 "delete ranges"，快速删除由于 DROP TABLE / DROP Index 等操作产生的整区间的垃圾数据。

第三阶段会扫描每个 TiKV 上的数据，针对每一个 key 删除其不再需要的老旧版本。会对每个 key 保留 safe point 前的最后一次写入（除非最后一次写入是删除）。只需要将 safe point 发送给 PD，即可结束整轮 GC。TiKV 会自行检测到 safe point 发生了更新，会对当前节点上所有作为 Region leader 进行 GC。与此同时，GC leader 可以继续触发下一轮 GC。

## Resolve lock 的实现

## GC 的具体执行过程

## GC 对 TiCDC 的影响

## GC 对 BR 的影响

## 参考

https://docs.pingcap.com/zh/tidb/dev/garbage-collection-overview