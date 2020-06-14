---
title: 偏向锁竟然与hashcode有关
date: 2019-07-04 23:24:22
tags: tingle
---

偶然间得知的一个知识点，在没有重写hashcode的情况下，如果调用了hashCode()方法，那么会导致偏向锁无法使用，如果你不了解技术内幕的话一定会觉得匪夷所思，hashcode和偏向锁有什么关系吗？
1.hashcode不做过多介绍了，应该大部分人都很了解了，如果不知道的建议随便搜索引擎查一下，重点说一下hashCode()方法，如果你没有重写Object类的hashCode方法的话那么会默认调用System.identityHashCode获取；
2.偏向锁是java为了优化synchronized同步锁而做的技术优化，当一个对象总是被同一线程获取，也就是说没有任何线程冲突的情况下，那么java自动会加上偏向锁，即永远偏向当前线程，每次会通过cas方法判断是否是同一线程在获取锁；
3.一个java对象的对象结构是由对象头，实例数据和对齐填充字节组成的，其中对象头又分为markword，指向类的指针和数组长度（数组对象才有）。

那么为什么偏向锁和hashcode会产生联系呢？其实很简单，在计算hashcode后会将hash值放入对象的markword头中，而恰恰偏向锁的实现逻辑也是将thread id放入markword头，这就导致了冲突。
下面是这个问题的一篇相关博客
<https://blogs.oracle.com/dave/biased-locking-in-hotspot>
不方便打开的话我将关于这个问题的英文摘出来了，都是很基础的单词就不翻译了，顺便说一句，英文对于程序员来说真的很重要，要有一定的paper阅读能力
Finally, there's not currently space in the mark word to support both an identity hashCode() value as well as the thread ID needed for the biased locking encoding. Given that, you can avoid biased locking on a per-object basis by calling System.identityHashCode(o). If the object is already biased, assigning an identity hashCode will result in revocation, otherwise, the assignment of a hashCode() will make the object ineligible for subsequent biased locking. This property is an artifact of our current implementation.