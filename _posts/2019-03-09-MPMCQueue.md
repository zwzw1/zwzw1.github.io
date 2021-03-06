---
layout: post
title: 'folly fbvector'
date: 2019-03-10
author: Zhu Wen
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: C++ folly
---

### 代码位置
MPMCQueue.h

### folly MPMCQueue

  // Note: Using CRTP static casts in several functions of this base
  // template instead of making called functions virtual or duplicating
  // the code of calling functions in the derived partially specialized
  // template
继承自MPMCQueueBase，使用CRTP避免了虚函数的开销
base中指定了common函数，在derived中实现这些函数，类似template方法
如下：
  template <typename... Args>
  bool write(Args&&... args) noexcept {
    uint64_t ticket;
    Slot* slots;
    size_t cap;
    int stride;
    if (static_cast<Derived<T, Atom, Dynamic>*>(this)->tryObtainReadyPushTicket(
            ticket, slots, cap, stride)) {
      // we have pre-validated that the ticket won't block
      enqueueWithTicketBase(
          ticket, slots, cap, stride, std::forward<Args>(args)...);
      return true;
    } else {
      return false;
    }
  }


多写者多读者的队列
  /// Enqueues a T constructed from args, blocking until space is
  /// available.  Note that this method signature allows enqueue via
  /// move, if args is a T rvalue, via copy, if args is a T lvalue, or
  /// via emplacement if args is an initializer list that can be passed
  /// to a T constructor.
blockingWrite


  /// If an item can be enqueued with no blocking, does so and returns
  /// true, otherwise returns false.  This method is similar to
  /// writeIfNotFull, but if you don't have a specific need for that
  /// method you should use this one.
  ///
  /// One of the common usages of this method is to enqueue via the
  /// move constructor, something like q.write(std::move(x)).  If write
  /// returns false because the queue is full then x has not actually been
  /// consumed, which looks strange.  To understand why it is actually okay
  /// to use x afterward, remember that std::move is just a typecast that
  /// provides an rvalue reference that enables use of a move constructor
  /// or operator.  std::move doesn't actually move anything.  It could
  /// more accurately be called std::rvalue_cast or std::move_permission.
  template <typename... Args>
  bool write(Args&&... args) noexcept {

write           只取决于slot是否可写
writeIfNotFull  使用pushTick和popTick来计算是否可写，并且会被阻塞


tryWriteUntil

read
blockingRead
isEmpty
tryReadUntil




  /// We assign tickets in increasing order, but we don't want to
  /// access neighboring elements of slots_ because that will lead to
  /// false sharing (multiple cores accessing the same cache line even
  /// though they aren't accessing the same bytes in that cache line).
  /// To avoid this we advance by stride slots per ticket.
  ///
  /// We need gcd(capacity, stride) to be 1 so that we will use all
  /// of the slots.  We ensure this by only considering prime strides,
  /// which either have no common divisors with capacity or else have
  /// a zero remainder after dividing by capacity.  That is sufficient
  /// to guarantee correctness, but we also want to actually spread the
  /// accesses away from each other to avoid false sharing (consider a
  /// stride of 7 with a capacity of 8).  To that end we try a few taking
  /// care to observe that advancing by -1 is as bad as advancing by 1
  /// when in comes to false sharing.
  ///
  /// The simple way to avoid false sharing would be to pad each
  /// SingleElementQueue, but since we have capacity_ of them that could
  /// waste a lot of space.
使用capacity 和stride来进行 step，避免cache false sharing
使用stride跳过cacheline的方式


turnSequencer
用来串行化同步的访问 ，相当于一个细粒度的锁


popTicket_  pop指针
pushTicket_ push指针

slot[] 存储数据
index = ((ticket * stride) % cap)
使用index进行访问，ticket是popTick或者pushTick，有stride是为了避免false sharing

每个slot都是一个单元素的block queue

turn表示第几轮，比如所有的从0~cap是第一轮，wrap以后是第二轮
比如：
!slots[idx(ticket, cap, stride)].mayEnqueue(turn(ticket, cap))
enqueue是偶数 (2 * turn)，dequeue是奇数( 2 * turn + 1)
当enqueue成功一个slot后，更新turn成2turn+1，这样dequeue就可以读取了
turn是读写的标志位
一个slot，如果不是空的，就不能写，写完以后，如果没被读过，就不能再写。这都是由turn的读写来控制的

mayEnqueue ，就是用来判断这位是否可写
pushTicket_ 是用来同步所有push thread的
popTicket_ 用来同步所有pop thread
tick用来获得是否可写可读slot的权利，slot中的turn用来决定是否读写完
为什么使用turn？简单地使用isWrite不可以么？
mpmcqueue像一个链表，每个slot独立的评估每个读写

dstate_ 含有一个seqLock位，写优先(tryExpand优先)
MPMCQueue以10为开始，10倍速的增长，直到capacity为止


  ClosedArray* closed_; 使用closed来保存已经关闭的buf，每当buf被expand后，原来的buf被保存在closed_中


    /// Dynamic state. A packed seqlock and ticket offset
  Atom<uint64_t> dstate_;
  seqlock用来expand的时候互斥写
  offset:
      uint64_t ticket = 1 + std::max(this->pushTicket_.load(), this->popTicket_.load());
      this->dstate_.store((ticket << kSeqlockBits) + (2 * (index + 1)));
      ticket即是offset
  如果ticket >= offset，表示是后面的访问
  如果ticket < offset，表示访问的是closed slot

tryObtainReadyPushTicket
如果slot无法写入，就表示写满了，即需要expand

Key points:
1. slot的顺序访问都是独立的cache line，用stride进行实现
2. 是否可以read,write，不用比较读写指针，简单的slot中就可以区分，这样就保证了不会每个访问都有查read point和write point，读写相互独立


如果queue只有一个写者，其实可以不用push_tick
如果只有一个读者，也可以不用pop_tick
push_tick和pop_tick分别是用来同步读者与读者，写者与写者的

