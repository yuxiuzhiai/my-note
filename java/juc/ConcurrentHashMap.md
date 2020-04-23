# 介绍(ConcurrentHashMap的源码注释中文版=。=)

ConcurrentHashMap是HashMap的并发安全版本，是HashTable的替代者。ConcurrentHashMap支持全并发的数据获取，以及高并发的数据更新（这里是相对于HashTable来说的，HashTable几乎所有的方法都加了synchronized关键字，也就是说无论是数据获取还是数据更新HashTable同时都支持一个线程）。

ConcurrentHashMap具备与HashTable相同的功能，但是数据获取通常并不需要加锁，且不支持锁全表，即使它是线程安全的。检索操作通常不会阻塞，因此可能与更新操作重叠。检索反映了最近发生的已完成更新操作的结果。更正式地说，给定键的更新操作happens-before 之后与该键相关的检索操作。

对于putAll()和clear()之类的聚合操作，并发检索可能仅反映某些条目的插入或删除。同样，迭代器，拆分器和枚举返回的元素反映了在创建迭代器/枚举时或此后某个时刻哈希表的状态。他们不会抛出ConcurrentModificationException 异常。但是，迭代器被设计为一次只能由一个线程使用，请记住，包括size(),isEmpty()和containsValue()方法在内的聚合状态方法的结果通常仅在映射未在其他线程中进行并发更新时才有用。否则，这些方法的结果将反映可能足以用于监视或估计目的但不适用于程序控制的瞬间状态。

与Hashtable类似，但与HashMap不同，ConcurrentHashMap不允许null用作键或值。

# 实现

1.8对ConcurrentHashMap的实现做了更改，1.8之前，ConcurrentHashMap是通过把内部的Entry数据分为Segment段，这样如果是操作不同Segment里的key，那多个线程就可以同时进行，不会阻塞彼此。

而1.8之后，ConcurrentHashMap的实现几乎跟之前是完全不一样，内部已经没有了Segment的概念了（其实还有，但是只是为了兼容老版本的序列化而已）

## 字段

* Node<K,V>[] table：跟HashMap一样，一个Node数据用于存储内容
* Node<K,V>[] nextTable：仅仅在rehash的时候不为null
* long baseCount：
* int sizeCtl：在table初始化和rehash的时候使用。当其实负值的时候，代表table正在初始化或者rehash。-1 代表初始化，如果是正在rehash，则值为-(1+rehash的线程数)
* int transferIndex
* int cellsBusy
* CounterCell[] counterCells
* KeySetView<K,V> keySet
* ValuesView<K,V> values
* EntrySetView<K,V> entrySet



## put(K k,V v)方法

```java
public V put(K key, V value) {
  //put方法以及putIfAbsent方法的统一入口
  return putVal(key, value, false);
}
```

那就进入putVal(K key,V value,boolean onlyIfAbsent)方法看一看：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
  if (key == null || value == null) throw new NullPointerException();
  int hash = spread(key.hashCode());
  int binCount = 0;
  for (Node<K,V>[] tab = table;;) {
    Node<K,V> f; int n, i, fh;
    if (tab == null || (n = tab.length) == 0)
      tab = initTable();
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
      if (casTabAt(tab, i, null,
                   new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
    }
    else if ((fh = f.hash) == MOVED)
      tab = helpTransfer(tab, f);
    else {
      V oldVal = null;
      synchronized (f) {
        if (tabAt(tab, i) == f) {
          if (fh >= 0) {
            binCount = 1;
            for (Node<K,V> e = f;; ++binCount) {
              K ek;
              if (e.hash == hash &&
                  ((ek = e.key) == key ||
                   (ek != null && key.equals(ek)))) {
                oldVal = e.val;
                if (!onlyIfAbsent)
                  e.val = value;
                break;
              }
              Node<K,V> pred = e;
              if ((e = e.next) == null) {
                pred.next = new Node<K,V>(hash, key,
                                          value, null);
                break;
              }
            }
          }
          else if (f instanceof TreeBin) {
            Node<K,V> p;
            binCount = 2;
            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                  value)) != null) {
              oldVal = p.val;
              if (!onlyIfAbsent)
                p.val = value;
            }
          }
        }
      }
      if (binCount != 0) {
        if (binCount >= TREEIFY_THRESHOLD)
          treeifyBin(tab, i);
        if (oldVal != null)
          return oldVal;
        break;
      }
    }
  }
  addCount(1L, binCount);
  return null;
}
```

