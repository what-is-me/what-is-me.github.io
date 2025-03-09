---
mathjax: true
title: '泛读笔记：《TiQuE: Improving the Transactional Performance of Analytical Systems for True Hybrid Workloads》'
date: 2025-03-03 17:46:13
tags:
    - Reading Note
    - DataBase
    - HTAP
category: "泛读笔记"
---
[Paper](https://www.vldb.org/pvldb/vol16/p2274-faria.pdf)

## 摘要

长期以来，事务一直是数据库管理中的一个关键问题，并且有大量的架构和算法来支持和实现它们。当前最先进的技术专注于存储管理，并与其设计紧密结合，例如，需要全新的引擎来支持混合事务分析处理 (HTAP) 等新功能。我们通过提出在查询语言（如 SQL）中实现事务逻辑的提案来解决这一挑战。这意味着我们的方法可以分层在现有的分析系统上，但事务快照的检索和更新事务的验证在服务器中运行，并且可以利用优化查询引擎的高级查询执行功能。我们在 MonetDB 上演示了我们的提案 TiQuE，并在保持分析查询良好性能的同时，将事务吞吐量平均提高了 500 倍，使其与最先进的 HTAP 系统具有竞争力。

<!-- more-->

## 背景

![HTAP Tradeoff](f1.png)

HTAP在AP和TP负载性能上存在tradeoff。
