## HashMap的存储结构

### 链表结构

### 红黑树结构

## HashMap的插入，修改过程

HashMap的插入最终会调用putVal

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果hash数组为空或者长度为0，调用resize为数组重新分配空间
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    //i = (n - 1) & hash n为hash表的长度，这个操作能够保证查找一定在数组范围内。
    //如果节点为空，创新一个新的节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
          //hash表有冲突，处理冲突
            Node<K,V> e; K k;
            //判断第一个Node是不是需要查找的node
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)//如果当前的节点是红黑树节点，调用它的插入方式
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //循环遍历当前位置
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {//如果链表后续为空，直接将新插入的值放在链表尾部
                        p.next = newNode(hash, key, value, null);
                        //链表长度大于8转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果有相同的key,遍历结束
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    //替换原来的值
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)//如果当前大小大于门限，门限原本是初始容量*0.75  
            resize();//进行两倍扩容。
        afterNodeInsertion(evict);
        return null;
    }
```

