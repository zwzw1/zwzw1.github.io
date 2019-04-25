---
layout: post
title: 'folly fbvector'
date: 2019-03-10
author: Zhu Wen
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: C++ folly
---

### 代码位置
concurrenthashmap.h

### folly concurrentHashMap

* Based on Java's ConcurrentHashMap
 *
 * Readers are always wait-free.
 * Writers are sharded, but take a lock.
 *
 * The interface is as close to std::unordered_map as possible, but there
 * are a handful of changes:
 *
 * * Iterators hold hazard pointers to the returned elements.  Elements can only
 *   be accessed while Iterators are still valid!
 *
 * * Therefore operator[] and at() return copies, since they do not
 *   return an iterator.  The returned value is const, to remind you
 *   that changes do not affect the value in the map.
 *
 * * erase() calls the hash function, and may fail if the hash
 *   function throws an exception.
 *
 * * clear() initializes new segments, and is not noexcept.
 *
 * * The interface adds assign_if_equal, since find() doesn't take a lock.
 *
 * * Only const version of find() is supported, and const iterators.
 *   Mutation must use functions provided, like assign().
 *
 * * iteration iterates over all the buckets in the table, unlike
 *   std::unordered_map which iterates over a linked list of elements.
 *   If the table is sparse, this may be more expensive.
 *
 * * rehash policy is a power of two, using supplied factor.
 *
 * * Allocator must be stateless.
 *
 * * ValueTypes without copy constructors will work, but pessimize the
 *   implementation.
 *
 * Comparisons:
 *      Single-threaded performance is extremely similar to std::unordered_map.
 *
 *      Multithreaded performance beats anything except the lock-free
 *           atomic maps (AtomicUnorderedMap, AtomicHashMap), BUT only
 *           if you can perfectly size the atomic maps, and you don't
 *           need erase().  If you don't know the size in advance or
 *           your workload frequently calls erase(), this is the
 *           better choice.



使用segments数组进行存储
   mutable Atom<SegmentT*> segments_[NumShards];
