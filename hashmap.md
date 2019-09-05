# hashmap与concurrenthashmap jdk1.8

## hashmap数据结构和原理
### 数据的存取
hashmap使用key-value存储数据，支持null key和null value，在hash函数能够均匀的将key-value均匀分散在每个桶(bucket)内的条件下，可以以常数时间执行get和put，根据key的hash值在数组中定位元素位置。对于迭代操作，时间正比于桶(bucket)的个数和key-value的个数，因此迭代操作的时间与负载因子相关。  
  
```java
transient Node<K,V>[] table;
```  
  
hashmap使用数组来存储key-value，当发生hash碰撞时，采用分离链表法来存储多个元素，当链表元素过多时，将链表转化为红黑树，当元素个数减少会将红黑树转为链表。数组长度即hashmap的容量，默认容量初始化为16，容量必须是2的n次方。  

### hash的计算和map容量问题
一般来说，hash掩码尽量取质数，也就数map的容量应为质数。质数能够减少hash函数不太好的时候的碰撞次数。但是如果hash能够做到完全分散的话，容量可以取任何值。因此hashmap中，为了数据操作更加高效，将容量取2的n次方，将hash函数做一定优化。
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
hash的计算调用了key的hashCode函数，然后将hash的高16位与低16位做异或运算。
  
容量的计算。给定目标容量，计算最接近的2的n次方作为数组大小。理解要点：5个位运算将n的最高位1的所有低位都置为1。
```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

### 重要属性
```java
    //存储key-value的数组
    transient Node<K,V>[] table;
    //map中存储的key-value个数
    transient int size;
    //数据结构的次数，包括两种：修改了key-value的个数（增、删）；rehash
	//用来避免在迭代过程中修改map，快速失败
    transient int modCount;

    //resize阈值 (capacity * load factor)
	//当threshold为0表示使用默认容量
    int threshold;

    //负载因子
    final float loadFactor;
```

### Node和TreeNode
数据存储在Node[]数组里，如何支持链表和红黑树？
Node实现了Map.Entry接口。TreeNode是LinkedHashMap.Entry的子类，LinkedHashMap.Entry是HashMap.Node的子类，TreeNode是Node的子类，所以可以支持链表和红黑树。
```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
	}
```
Node存储了hash值，并且保存了下一个节点的引用。
```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```
LinkedHashMap.Entry加上了前后节点的引用。
```java
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
	}
```
TreeNode加上了父、左、右节点的引用和节点的红黑标记。
## get
调用getNode获取存储key的节点，如果不存在该节点(e == null)，返回null，否则返回该节点的值。该函数返回null有两种情况：没有该key的key-value；key的value为null。
```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```
由于数组长度是2的n次幂，节点在数组中的位置可以通过位运算(n - 1) & hash取得。getNode首先根据hash查找节点在数组中的位置。如果节点存在，会首先检查第一个节点。如果第一个元素不是要查找的节点，会判断是否有下一个节点。如果有下一个节点，会根据该节点是链表还是树继续查找。
为什么检查第一个元素？为什么要判断下一个节点是否为null？为什么不当`first = tab[(n - 1) & hash]) != null时`直接判断first节点是树还是链表直接查询？
  
hash能够将元素较好的分散并且负载因子选取合适的情况下，每个bucket中只有一个节点或没有节点，bucket中节点多于两个的情况概率较低，所以首先检查第一个节点，在检查下一个节点是否为null。
```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
		//根据哈希查找节点位置
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
			//首先检查第一个节点，如果存在该节点，大部分情况下到这里就返回了
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
			//如果不存在该节点，大部分情况到这里也返回了
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
## put
put调用了putVal插入key-value。
putVal要做几件事情：
1. table数组是懒惰初始化的，只有在第一次插入时才会为table数组分配内存。如果需要初始化数组会先初始化数组。
2. 找到插入位置，将值插入，在table数组中的位置是`i = (n - 1) & hash`。有几种情况要考虑：
2.1 table[i]位置为null可以直接插入。
2.2 如果table[i]不为null，那么table[i]是树还是链表？key是否存在？
2.3 在链表中插入了元素之后，要不要把链表转化为红黑树？
3. 要不要修改modCount？
4. 返回值。返回该key原先的值，如果没有返回null。

代码类似get也有优化，将概率较大的if放在了前面。带着putVal要做的几件事情代码比较容易理解。

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//判断是否需要初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//判断是否为null，是的话直接插入。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
		//查找key的位置
        else {
            Node<K,V> e; K k;
			//检查第一个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
			//树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
			//链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
			//e != null意味着存在这个key，直接替换值，肯定不需要修改modCount，不需要resize
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
		//如果能走到这里，意味增加了一个key-value而不是修改了value，增加modCount，看是否需要resize
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

  
**问题1 链表转红黑树**
链表插入中，用binCount记录该bucket下节点个数，当binCount大于TREEIFY_THRESHOLD，调用treeifyBin，这个函数并不一定会将链表转为红黑树，而是进行进一步判断。只有当table容量大于等于MIN_TREEIFY_CAPACITY并且binCount大于等于TREEIFY_THRESHOLD，才会转为红黑树，否则调用resize。
```java
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```
**问题2 modCount的修改**  
modCount在修改了key-value的个数或者rehash才会加1。如果能够查找到key，那么肯定没有增加key-value的个数，不需要修改。table[i]为null或者key不存在，才需要修改。

**问题3 什么时候resize**  
有三个地方会resize。resize不会增加modCount。
1. 初始化table调用了resize。
2. 调用treeifyBin时，有可能会resize。
3. 插入key-value后，size加1，如果比threshold大，会resize。

## remove  
与put类似，调用removeNode删除key-value。同样，该方法返回原先的value，返回null也无法判断原先是否存在该key-value。  
removeNode根据key和hash查找node的位置，只有key存在时才会删除。如果删除了key-value，则对key-value个数做了修改，需要修改modCount。binCount过少会unTreeifyBin。  
```java
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
			//如果查找到了key，则开始删除，分链表删除和树删除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```
**问题 什么时候树转化为链表**  
如果是树节点，调用`((TreeNode<K,V>)node).removeTreeNode(this, tab, movable)`删除该节点。这里的转化和`UNTREEIFY_THRESHOLD`无关。

## 扩容

