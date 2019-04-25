---
layout: post
title: 'folly fbvector'
date: 2019-03-10
author: Zhu Wen
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: C++ folly
---

### 代码位置
FBVector.h

### folly atomicUnorderedMap


findOrConstruct
主要使用slots_ 进行存储
插入时，如果slot被占用，则往下一个开始找，直到找到可用的slot，使用next将它们连起来，用于冲突检测
每个slot的index使用最后两位作为chain的status，就意味着每个bucket预留了4个空间


