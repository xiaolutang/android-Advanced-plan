# TreeMap

TreeMap是JAVA集合中的一种，它能够通过Comparable或自己在构造的时候提供Comparator来进行自然排序。

## TreeMap元素的添加修改过程

TreeMap#put

```java
public V put(K key, V value) {
        TreeMapEntry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new TreeMapEntry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        TreeMapEntry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        TreeMapEntry<K,V> e = new TreeMapEntry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
```

在不考虑如何进行存储的情况下添加数据的逻辑非常简单。

1. 如果提供了Comparator对象，则使用Comparator进行比较添加。如果没有提供则使用compareTo进行比较添加
2. 默认情况下TreeMap的比较值越小排名越靠前。
3. compareTo 返回负数说明当前对象的key小，返回正数说明当前Key大。0说明相等
4. 与第二条相识 Comparator#compare返回负数说明第一个对象的key小，返回正数说明第一个对象Key大。0说明相等
5. 虽然TreeMap通过红黑树来进行数据存储，但是TreeMapEntry的left,right构成了一个有序链表。
6. 添加的时候如果原来有同样的 Key那么新的value会替换原来的value

## TreeMap元素查找过程

TreeMap查找最终会调用getEntry方法

```java
final TreeMapEntry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        TreeMapEntry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }
```

可以看到查找的过程是遍历链表对key进行比较。

## TreeMap的删除过程

因为涉及到红黑树的数据结构，等待学习了相关的数据结构再来阅读源码。

## TreeMap如何比较相同

TreeMap通过compareTo函数或者外部提供的 Comparator对象比较是否相同，对插入的 元素进行升序排列。