# **ThreadLocal**

## 简介

从名称我们可以看到ThreadLocal叫做**线程变量**。ThreadLocal中填充的变量值属于**当前线程**，该变量对于其他线程而言是**隔离**的。ThreadLocal为变量在每一个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

## 使用方式

ThreadLocal的作用是为当前线程创建一个副本，如：

```
public static void main(String[] args) {
        ThreadLocal l = new ThreadLocal();
        l.set("1");
        ThreadLocal l1 = new ThreadLocal();
        l1.set("2");
        System.out.println(l.get());
        System.out.println(l1.get());
    }
```

输出结果： 

```
1
2
```

## 源码分析

ThreadLocal开始的例子使用的是get和set方法。[如果嫌弃太繁琐或者懒得看，可直接跳到总结](#2)

### set方法分析

#### set方法

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

从开始通过Thread.currentThread 获取当前的线程对象，通过传入线程对象获取ThreadLocalMap，不为空直接设置值，map中的key是**当前的ThreadLocal对象**，value是传入的值。

#### getMap 方法

```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

#### createMap方法

```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

从getMap 和 createMap可以看出获取的都是**Thread对象**中的**threadLocals这个成员变量**（类型是ThreadLocalMap）。在getMap中获取 Thread t ，t中的threadLocals。在createMap中对Thread t的threadLocals进行创建

#### Thread 类中的threadLocals

```
public class Thread implements Runnable {

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}

```

那么 **ThreadLocalMap**类型的threadLocals是什么呢？

```
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            // threadLocal 做key，value做值
        }
        .......
}
```

从代码中可以看出

ThreadLocalMap其实就是ThreadLocal的一个**静态内部类**，里面定义了一个Entry来保存数据，而且还是继承的[**弱引用**]()。在Entry内部使用ThreadLocal作为key，使用我们设置的value作为值。

### get方法分析

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
 ```

通过thread.currentThread方法获取当前的线程对象，作为Key从ThreadLocalMap获取对应的值。**setInitialValue**方法是如果没有获取到对应的值证明没有线程副本，进行null的线程副本的初始化(就是对ThreadLocalMap设置一个null值)。

### **remove方法**

```
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

就是通过线程对象获取对应的map，并将map中对应的key-value进行移除。

### <span id="2">ThreadLocal 已上代码总结</span>

* 每一个Thread线程维护着一个ThreadLocalMap引用
* ThreadLocalMap是ThreadLocal的内部类型，用Entry来进行存储。 继承的[**弱引用**]()。如果使用完threadLocal，不进行remove会出现[内存泄漏](#3)
* ThreadLocal创建的副本是存储在自己的threadLocals中的，也就是自己的ThreadLocalMap
* ThreadLocalMap的键值为ThreadLocal对象，而且可以有多个threadLocal变量，因此保存在map中
* 在进行get之前，必须先set，否则会报空指针异常，当然也可以初始化一个，但是必须重写initialValue()方法
* ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取value。

## <span id="3">内存泄漏</span>
![avatar]()
