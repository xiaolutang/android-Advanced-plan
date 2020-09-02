HashTable和HashMap一样是通过数组加链表的形式来存储数据。其增删改查基本和HashMap一致，不同的是HashTable通过synchronized来锁住整个表来实现线程安全的访问。

## HashTable和HashMap插入计算数组下标index的方式不同

HashTable : index = (插入key的hash & 0xffffffff) % 数组长度

HashMap: index 

## 扩容过程中对链表中数据的迁移过程不同。

HashTable通过计算index方式直接将链表表头进行替换，

hashMap通 hash & 原来表的长度 是否为0 构建两个链表来进行数据迁移。



## ConcurrentHashMap

ConcurrentHashMap将Hash表分为16桶(segment)，每次只对需要的桶进行加锁

## Collections.synchronized XXX map

通过包装的方式添加额外的职责，性能较低。