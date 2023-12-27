---
title: 解决CAS机制中ABA问题的AtomicStampedReference
---

AtomicStampedReference是一个带有时间戳的对象引用，能很好的解决CAS机制中的ABA问题，这篇文章将通过案例对其介绍分析。

**一、ABA问题**

ABA问题是CAS机制中出现的一个问题，他的描述是这样的。我们直接画一张图来演示，

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/4bed2e738bd4b31cce48475ac556867a9f2ff865.jpeg)

什么意思呢？就是说一个线程把数据A变为了B，然后又重新变成了A。此时另外一个线程读取的时候，发现A没有变化，就误以为是原来的那个A。这就是有名的ABA问题。ABA问题会带来什么后果呢？我们举个例子。

一个小偷，把别人家的钱偷了之后又还了回来，还是原来的钱吗，你老婆出轨之后又回来，还是原来的老婆吗？ABA问题也一样，如果不好好解决就会带来大量的问题。最常见的就是资金问题，也就是别人如果挪用了你的钱，在你发现之前又还了回来。但是别人却已经触犯了法律。

如何去解决这个ABA问题呢，就是使用今天所说的AtomicStampedReference。

**二、AtomicStampedReference**

1、问题解决

我们先给出一个ABA的例子，对ABA问题进行场景重现。

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/cc11728b4710b91222c13818817d5d069345225e.jpeg)

在上面的代码中，我们使用张三线程，对index10->11->10的变化，然后李四线程读取index观察是否有变化，并设置新值。运行一下看看结果：

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/8718367adab44aed59eb3ed2f19c2604a08bfbe1.jpeg)

这个案例重现了ABA的问题场景，下面我们看如何使用AtomicStampedReference解决这个问题的。

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/4610b912c8fcc3cef1bf4968d0c5778dd53f20f7.jpeg)

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/3801213fb80e7bec8279f0626dae183d9a506b1c.jpeg)

上面的代码我们再来分析一下，我们会发现AtomicStampedReference里面增加了一个时间戳，也就是说每一次修改只需要设置不同的版本好即可。我们先运行一边看看：

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/bba1cd11728b4710a7fc0eac814e62f8fd032300.jpeg)

这里使用的是AtomicStampedReference的compareAndSet函数，这里面有四个参数：

compareAndSet(V expectedReference, V newReference, int expectedStamp, int newStamp)。

（1）第一个参数expectedReference：表示预期值。

（2）第二个参数newReference：表示要更新的值。

（3）第三个参数expectedStamp：表示预期的时间戳。

（4）第四个参数newStamp：表示要更新的时间戳。

这个compareAndSet方法到底是如何实现的，我们深入到源码中看看。

2、源码分析

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/b21c8701a18b87d6f4f4c1064588893d1e30fdff.jpeg)

刚刚这四个参数的意思已经说了，我们主要关注的就是实现，首先我们看到的就是这个Pair，因此想要弄清楚，我们再看看这个Pair是什么，

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/48540923dd54564ecfaab273f15e3d87d0584f36.jpeg)

在这里我们会发现Pair里面只是保存了值reference和时间戳stamp。

在compareAndSet方法中最后还调用了casPair方法，从名字就可以看到，主要是使用CAS机制更新新的值reference和时间戳stamp。我们可以进入这个方法中看看。

![img](/images/解决CAS机制中ABA问题的AtomicStampedReference/a8773912b31bb05182752a8075fa7bb14bede04f.jpeg)

三、总结

其实除了AtomicStampedReference类，还有一个原子类也可以解决，就是AtomicMarkableReference，它不是维护一个版本号，而是维护一个boolean类型的标记，用法没有AtomicStampedReference灵活。因此也只是在特定的场景下使用。