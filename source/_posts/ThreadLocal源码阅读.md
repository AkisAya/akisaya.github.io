---
title: ThreadLocal 源码阅读(JDK8)
date: 2018-11-01 11:41:30
categories: java
tags:
- 源码
- java基础
---

> ThreadLocal 为线程提供了线程本地变量，不同于其他的变量，线程本地变量是通过 get() 和 set() 方法。ThreadLocal 通常是一个私有静态域，与Thread中的某个状态相关联（如：UserID 或者 TransactionId）

ThreadLocal 能够用来实现多线程中数据的隔离，避免不必要的并发控制的麻烦。

<!--more-->

## ThreadLocal 的基本使用
```
public class ThreadLocalTest {
    public static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 1;
        }
    };

    public static void main(String[] args) {
        System.out.println(threadLocal.get());
        threadLocal.set(5);
        System.out.println(threadLocal.get());
    }
}
```
ThreadLocal 是泛型的。除了 ThreadLocal 的 get() 和 set() 和 remove() 方法外，在实例化一个 ThreadLocal 的时候，还可以选择覆盖 initialValue() 方法提供一个初值。

## 源码分析
### ThreadLocal
那么 ThreadLocal 是怎么实现的呢?
每个 Thread 都含有一个采用线性探测的Hash Map，这个 Map 不是 JDK 中提供的 HashMap，而是定义在 ThreadLocal 中的一个静态内部类 ThreadLocalMap。
来看一看 ThreadLocal 中 get() 和 set() 方法
```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```
可以看到在 get() 方法中先获取当前线程，然后得到当前线程的 Map 对象，以 ThreadLocal 这个对象本身为 key 取出值。如果得到的 map 为空，说明是第一次访问这个方法并且在之前没有执行过 set() 方法，那么就返回类在实例化时覆盖的 initialValue() 方法的值。
```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
getMap() 方法返回了 Thread 的 threadLocals 变量，说明这个 Map 对象是定义在 Thread 中的。
```
// Thread.java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```
在上述的 set() 和 setInitialValue() 方法中可以看到如果 Map 还没有创建则创建 Map 对象。
```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
在这个方法中，完成了 Thread 的 threadLocals 的实例化。

### ThreadLocalMap
ThreadLocalMap 是定义在 ThreadLocal 中的静态内部类，它是一种 Hash 的 Map，以 ThreadLocal 为 Key。但是 并不是用的继承自 Object 对象的 hashcode() 方法产生 hash 值。
```
private final int threadLocalHashCode = nextHashCode();

/**
 * The next hash code to be given out. Updated atomically. Starts at
 * zero.
 */
private static AtomicInteger nextHashCode =
    new AtomicInteger();

/**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */
private static final int HASH_INCREMENT = 0x61c88647;

/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
每个 ThreadLocal 都有一个 threadLocalHashCode 与其绑定，并且通过一个静态的原子类，保证了每个类的 threadLocalHashCode 都不相同。此外还加上了一个静态常量，按注释看是~~将这个 hashcode 变成一个二次方表中的近最优扩展的乘法哈希值~~（turns implicit sequential thread-local IDs into near-optimally spread multiplicative hash values for power-of-two-sized tables，大误，猜测是为了优化后续的hash冲突的）。


下面来看一看 ThreadLocalMap
> ThreadLocalMap is a customized hash map suitable only for maintaining thread local values. No operations are exported outside of the ThreadLocal class. The class is package private to allow declaration of fields in class Thread.  To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys. However, since reference queues are not used, stale entries are guaranteed to be removed only when the table starts running out of space.

ThreadLocalMap 是一个静态内部类，只是为了给 ThreadLocal 提供辅助而不对外提供接口，同时每一 Entry 的键使用了弱引用，会导致键的垃圾回收。这种键被垃圾回收的 Entry 就是一个旧的条目，虽然键为空，但是 Value 仍可能指向一个对象。旧条目会在 table 容量不足的时候被清理。
```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
以上为 Entry 的定义，ThreadLocal 对象作为一个 Key 被包装成了一个虚引用。

前文提到了 ThreadLocalMap 是采用了线性探测的 Hash Map，所以内部是维护了一个数组用来存放键值对。初始值为 16, 容量必须为 2 的倍数，因为 2^n - 1 的二进制表示全是 1 (15 = 0b1111)，再得到 Entry 在数组中的索引的时候，保证刚好是 <= INITIAL_CAPACITY 的
```
/**
 * The initial capacity -- MUST be a power of two.
 */
private static final int INITIAL_CAPACITY = 16;

/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
详细看下 ThreadLocalMap 的 get 和 set 方法
```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```
一开始假定可以直接命中，直接命中的话就直接返回，没有命中的话再委托给 getEntryAfterMiss， 这个方法会进行线性探测。
```
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
             (i); // 删除陈旧的条目，并重新组织数据
        else
            i = nextIndex(i, len); // 探测
        e = tab[i];
    }
    return null;
}
```
```
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 如果已经包含 key, 则重置 value
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果 key 为空，说明 key 被垃圾回收了，这个时候替换掉这个 key
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 在空调目处新建一个条目
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash(); // 如果超过了阈值就 resize 然后重新 hash
}
```
set 方法如上所示，注释已经过程比较明白了。因为每个 Entry 的 Key 是一个虚引用，所以存在被 GC 的可能。
此外还有 remove 方法，要注意的是删除后要重新hash。其他还有如 expungeStaleEntry 等方法都是删除陈旧条目或者是rehash。



### 其他
JDK1.8 中新增了 Suppier ，用来提供 lambda 表达式调用
