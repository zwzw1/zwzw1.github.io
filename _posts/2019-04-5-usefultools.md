---
layout: post
title: 'folly fbvector'
date: 2019-03-10
author: Zhu Wen
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: C++ folly
---

### 代码位置


### folly tools

#define SCOPE_EXIT
#define SCOPE_EXIT                               \
  auto FB_ANONYMOUS_VARIABLE(SCOPE_EXIT_STATE) = \
      ::folly::detail::ScopeGuardOnExit() + [&]() noexcept

ThreadCachedInt

seqlock
futex

