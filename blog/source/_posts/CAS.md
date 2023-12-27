title: CAS

## CAS简介

CAS指的是现代CPU广泛支持的一种对内存中的共享数据进行操作的一种特殊指令。这个指令会对内存中的共享数据做原子的读写操作。在Java并发应用中通常指CompareAndSwap或CompareAndSet，即比较并交换，是实现并发算法时常用到的一种技术。java.util.concurrent包中借助CAS实现了区别于synchronized同步锁的一种乐观锁。乐观锁就是每次去取数据的时候都乐观的认为数据不会被修改，因此这个过程不会上锁，但是在更新的时候会判断一下在此期间的数据有没有更新.

## CAS思想

CAS有三个参数，当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做，并返回false.

## CAS优缺点

1. 系统在硬件层面保证了CAS操作的原子性，不会锁住当前线程，它的效率是很高的。但是在并发越高的条件下，失败的次数会越多，CAS如果长时间不成功，会极大的增加CPU的开销，因此CAS不适合竞争十分频繁的场景
2. CAS只能保证一个共享变量的原子操作，对多个共享变量操作时，无法保证操作的原子性，这时就可以用锁，或者把多个共享变量合并成一个共享变量来操作。JDK提供了AtomicReference类来保证引用对象的原子性，可以把多个变量放在一个对象里来进行CAS操作.

## 原子方式更新数组

```
public class Main {

    static int[] valueArr = new int[]{1, 2};

    //AtomicIntegerArray内部会拷贝一份数组
    static AtomicIntegerArray ai = new AtomicIntegerArray(valueArr);

    public static void main(String[] args) {
        ai.getAndSet(0, 3);
        //不会修改原始数组value
        System.out.println(ai.get(0));
        System.out.println(valueArr[0]);
    }
}
```

## 原子方式更新引用

```
public class Main {

    public static AtomicReference<User> atomicUserRef = new AtomicReference<User>();

    public static void main(String[] args) {
        User user = new User("Jack", 22);
        User updateUser = new User("Rose", 20);

        atomicUserRef.set(user);
        atomicUserRef.compareAndSet(user, updateUser);

        System.out.println(atomicUserRef.get().getName());
        System.out.println(atomicUserRef.get().getOld());
    }
    static class User {
        private String name;
        private int old;

        public User(String name, int old) {
            this.name = name;
            this.old = old;
        }

        public String getName() {
            return name;
        }

        public int getOld() {
            return old;
        }
    }
}
```

## **原子方式更新字段**

```
public class Main {

    private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater
            .newUpdater(User.class, "old");

    public static void main(String[] args) {
        User user = new User("Hensen", 20);
        System.out.println(a.getAndIncrement(user));
        System.out.println(a.get(user));
    }

    public static class User {
        private String name;
        public volatile int old;//注意需要用volatile修饰

        public User(String name, int old) {
            this.name = name;
            this.old = old;
        }

        public String getName() {
            return name;
        }

        public int getOld() {
            return old;
        }
    }
}
```

