LinkedHashMap继承自HashMap许多不同之处在于LinkedHashMap能够根据插入或访问顺序来存储数据。

我们知道 hashMap在存储数据的时候通过newNode方法生成一个Node节点保存相关的数据.。Node的数据结构如下：

```java
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
 }
```

LinkedHashMap为了保存插入顺序和访问顺序的相关信息，在这个node数据的基础上新增了前驱后继。

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

同时LinkedHashMap保存了链表的头节点和尾结点

```java
/**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> tail;
    
```

每次遍历都是从头结点到尾结点，也就是说在默认情况下先插入的数据会被优先访问

## LinkedHashMap如何保证按插入顺序访问

LinkedHashMap重写了newNode方法，每次生成新的节点都会调用linkNodeLast。再双向链表的尾部插入数据

```java
 // link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;//保存链表尾部的信息
        tail = p;//将本次插入节点的引用赋值给链表表尾
        if (last == null)//如果上次表尾为空，说明双向链表为空，插入的节点既是表头也是表尾
            head = p;
        else {//将原来的表尾和新插入的数据进行双向连接
            p.before = last;
            last.after = p;
        }
    }
```



## LinkedHashMap如何保证按访问顺序排序

想要按照访问顺序来排序，也就是每次插入、修改、访问的节点需要放在头结点处。

想要按照访问顺序访问LinkedHashMap中的数据。需要将成员变量accessOrder设置成true。

还有一点需要注意，linkedHashMap是否需要移除元素需要配合removeEldestEntry方法来实现，但是默认返回的是false因此。如果用linkedhashmap来实现它。我们需要根据自己的逻辑来实现它。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```



### 替换元素时移动到链表位置

HashMap在插入数据的时候如果原来已经存在这个数据，会用新的value替换原来的value并调用 afterNodeAccess。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

上面代码的逻辑非常简单，将新的节点直接放在**链表的尾部**，这个和我们常规想象有一点出入，原来我以为是放在链表的头部。

### 访问元素移动到链表尾部

LinkedHashMap重写了get方法。

```java
 public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);//同样会调用这来移动元素到链表尾部
        return e.value;
    }
```



## 问题：如何利用LinkedHashMap实现Lru功能。

将LinkedHashMap的accessOrder设置为true,在put元素之后检查长度是不是超过指定范围，如果超过移除链表表头的节点。