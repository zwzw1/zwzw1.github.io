---
layout: post
title: 'folly fbvector'
date: 2019-03-10
author: Zhu Wen
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: C++ folly
---

### 代码位置
atomicHashMap.h
atomicHashMap-inl.h

### folly atomicHashMap


/*
 * AtomicHashMap --
 *
 * A high-performance concurrent hash map with int32 or int64 keys. Supports
 * insert, find(key), findAt(index), erase(key), size, and more.  Memory cannot
 * be freed or reclaimed by erase.  Can grow to a maximum of about 18 times the
 * initial capacity, but performance degrades linearly with growth. Can also be
 * used as an object store with unique 32-bit references directly into the
 * internal storage (retrieved with iterator::getIndex()).
 *
 * Advantages:
 *    - High-performance (~2-4x tbb::concurrent_hash_map in heavily
 *      multi-threaded environments).
 *    - Efficient memory usage if initial capacity is not over estimated
 *      (especially for small keys and values).
 *    - Good fragmentation properties (only allocates in large slabs which can
 *      be reused with clear() and never move).
 *    - Can generate unique, long-lived 32-bit references for efficient lookup
 *      (see findAt()).
 *
 * Disadvantages:
 *    - Keys must be native int32 or int64, or explicitly converted.
 *    - Must be able to specify unique empty, locked, and erased keys
 *    - Performance degrades linearly as size grows beyond initialization
 *      capacity.
 *    - Max size limit of ~18x initial size (dependent on max load factor).
 *    - Memory is not freed or reclaimed by erase.
 *
 * Usage and Operation Details:
 *   Simple performance/memory tradeoff with maxLoadFactor.  Higher load factors
 *   give better memory utilization but probe lengths increase, reducing
 *   performance.
 *
 * Implementation and Performance Details:
 *   AHArray is a fixed size contiguous block of value_type cells.  When
 *   writing a cell, the key is locked while the rest of the record is
 *   written.  Once done, the cell is unlocked by setting the key.  find()
 *   is completely wait-free and doesn't require any non-relaxed atomic
 *   operations.  AHA cannot grow beyond initialization capacity, but is
 *   faster because of reduced data indirection.
 *
 *   AHMap is a wrapper around AHArray sub-maps that allows growth and provides
 *   an interface closer to the STL UnorderedAssociativeContainer concept. These
 *   sub-maps are allocated on the fly and are processed in series, so the more
 *   there are (from growing past initial capacity), the worse the performance.
 *
 *   Insert returns false if there is a key collision and throws if the max size
 *   of the map is exceeded.
 *
 *   Benchmark performance with 8 simultaneous threads processing 1 million
 *   unique <int64, int64> entries on a 4-core, 2.5 GHz machine:
 *
 *     Load Factor   Mem Efficiency   usec/Insert   usec/Find
 *         50%             50%           0.19         0.05
 *         85%             85%           0.20         0.06
 *         90%             90%           0.23         0.08
 *         95%             95%           0.27         0.10

默认构造函数在atomicHashArray.h中定义
template <
    class KeyT,
    class ValueT,
    class HashFcn = std::hash<KeyT>,
    class EqualFcn = std::equal_to<KeyT>,
    class Allocator = std::allocator<char>,
    class ProbeFcn = AtomicHashArrayLinearProbeFcn,
    class KeyConvertFcn = Identity>
class AtomicHashMap;

使用atomicHashArray作为底层结构

冲突检测，使用一个与冲突数有关的函数
    idx = ProbeFcn()(idx, numProbes, capacity_);
实现是在atomichashMap-inl.h中
atomicHashMap.h只是声明

  typedef AtomicHashArray<
      KeyT,
      ValueT,
      HashFcn,
      EqualFcn,
      Allocator,
      ProbeFcn,
      KeyConvertFcn>
      SubMap;
  std::atomic<SubMap*> subMaps_[kNumSubMaps_];  存放所有的items
  subMaps_[0] 是primary map
  其它的subMaps就用来对capacity进行扩充的。如果subMaps_[0]找不到，就去找subMaps_[ i ]
  相当于是一个二维数组，即是：atomicHashArray[ i ]

  template <class ContT, class IterVal, class SubIt>
  struct ahm_iterator;

1. insert的实现是，先lock住，再插入，可以保证不会冲突
2. find就是按照subMap的顺序查找
3. erase只是把key设置成erased状态，而不改变value，就保证了多线程的访问。比如一个线程开始访问value，另一个erase吧这个key，前面一个线程的value也不会有错。
4. update 是可以冲突的


