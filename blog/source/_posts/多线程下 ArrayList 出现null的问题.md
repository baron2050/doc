---
title: 多线程下 ArrayList 出现null的问题
---

发现这个问题
在某个项目中使用了ArrayList 了,将他带入到 子线程中去添加待定值,然后出现了意向不到的错误,报空指针异常,出现一个 null 值,而且该问题不必现,有时候经常跑代码才出现几次.

排查
反复查看代码,未发现明显可能会出现 null 的地方,返回的值(上文中的添加待定值)在其他方法中都不为null
在代码中插入打印代码判断为null的情形,进行打印,加断点,分析,发现 在对添加待定值 进行为null 判断时不起作用,而 在对list 进行为null 判断时进入断点,
结合这个反馈和代码的源码进行分析,感觉可能是ArrayList 的问题,这才突然想起来ArrayList 是线程不安全的,这才百度发现确实有这个问题
代码进行验证
在大插入下,运行 3-4 次就有一次会出现 null 的情况,证实了想法

```java
// ArrayList 线程不安全, add 方法在多线程 环境下 会出现意外的 null 值
@Test
void testListThread() throws InterruptedException {
    ArrayList list = new ArrayList();
    CountDownLatch latch = new CountDownLatch(1000);
   for (int i = 0; i < 1000; i++) {
       new Thread(new Runnable() {
           @Override
            public void run() {
                list.add("abc");
               latch.countDown();
           }
        }).start();
  }

    latch.await();
    if(list.contains(null)) {
        System.out.println("list 包含 null 值");
        int index = list.indexOf(null);
        System.out.println(index);
        System.out.println(list);
    }

}
```

反省

对java 集合的源码阅读还是不够熟悉,有些知识明明之前看过,但还是习惯性的用哪些常用的线程不同步的类,在多线程下没用仔细思考选用的对象
解决方案

1. List list = Collections.synchronizedList(new ArrayList(...)); 使用 Collections.synchronizedList 去包裹要使用的集合
2. CopyOnWriteArrayList（线程安全,但是可能会对内存占用较大,垃圾回收频繁)
3. Vector (线程同步的集合对象,操作大数据的时候会比较慢)