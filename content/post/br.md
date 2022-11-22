---
title: "Br"
date: 2022-11-21T17:21:49+08:00
lastmod: 2022-11-21T17:21:49+08:00
draft: false
keywords: ["br", "tidb"]
description: "walk through tidb br' source code"
tags: ["tidb", "br"]
categories: ["development"]
author: "Ling Jin"

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

1. br 的应用场景
2. 什么是全量备份

全量备份是对集群某个时间点的全量数据进行备份。

3. 什么是 PITR


# Backup 工作过程

```
tiup br backup db --db=sysbench --pd=http://10.2.7.4:2379 --storage="s3://db-sysbench-300?access-key=minioadmin&secret-access-key=minioadmin&endpoint=http%3a%2f%2f10.2.7.72:9199&force-path-style=true" --send-credentials-to-tikv=true
```

Backup 执行的产物：

1. backup.lock
2. backupmeta
3. 多个文件夹，每个文件夹里存放多个 sst 文件。


Backup 的基本工作过程：

1. 初始化到下游备份存储节点的连接
2. 找到需要被备份的 ranges
3. 根据 backup-ts，拿到一个 snapshot，在该 snapshot 智商，对目标 ranges 进行备份。



备份数据，备份 schema，写 meta。

对每一个 schema 计算 checksum。发送 checksum request 到 tikv，收集 response，构建 checksum 结果。


（添加流程图）大致流程：

检查配置 -> 创建 client -> 创建 backup storage -> 检查 backup storage 的可用性，不能有其他 backup 正在使用该 storage -> 对 backup storage 上 file lock -> 修改 gc ttl -> 生成 backup ts

启动 safe point keeper，周期性更新 gc safe point。

创建 `backupRequest`, StartVersion <-> EndVersion。


## meta writer 的工作过程

创建 meta writer。

`FlushBackUpMeta` 将 meta 序列化，然后加密，再写到目标存储系统。


## Backup ranges

`client.BackupRanges`，按照 range 备份，并行执行。分别调用 `client.BackupRange`。

1. 找到所有的 tikv store
2. 将 backup requst 发送到所有 tikv instances (stores)，收集对应的 response。
3. 执行 fineGrainedBackup

根据步骤 2 手机到的结果，构建已经被备份的 ranges。查看是否存在尚未被备份的 ranges，如果存在则对这些 ranges 执行 backup 操作。


## Backup schemas

## Flush meta


backup ts 是什么。基于 TiDB MVCC 实现的快照备份，backup-ts 指定了备份的 tso。

备份存储地址

告知 tikv 对目标 regions 进行备份

tikv 侧的 backup worker

数据备份 -> 元信息备份

是否备份索引等数据？

如何对备份数据进行压缩？ztsd


前置检查，连接 pd，tikv

拿到需要被备份的 ranges，schemas，placement polies





`metaWriter.FinishWriteMetas`

备份 schemas：对每个 schema 计算 checksum，将 table 的统计信息导出到 json。

schema 是什么？









备份过程中，执行了 DDL 怎么办？

备份过程中，发生了 region 变化怎么办？




Full

DB

Table

Raw


# Restore 过程

# PITR