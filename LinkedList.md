# LinkedList
`LinkedList`实现了`List<E>`, `Deque<E>`, `Cloneable`, `java.io.Serializable`接口，可以作为List和双端队列使用，`Cloneable`, `java.io.Serializable`是起指示作用的接口，表示可以克隆和序列化。
```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

## 几个关键属性
`LinkedList`使用双向链表存储元素，节点为
```
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
```
    //size,元素个数
    transient int size = 0;
    //头结点
    transient Node<E> first;
    //尾结点
    transient Node<E> last;

```

## 构造函数
提供了默认构造函数和用集合来构造List。由于不需要数组复制操作，因此不需要像`ArrayList`一样预先分配内存。
```
    public LinkedList() {
    }
```
```
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

## 链表操作
`LinkedList`作为`List`接口和`Deque`接口的实现类，这些主要操作都是通过以下几个函数实现的：
```
void linkFirst(E)   //在链表前面插入元素
void linkLast(E)    //在链表最后插入元素
void linkBefore(E, Node<E>)   //在Node<E>前面插入元素
E unlinkFirst(Node<E>)    //删除第一个元素
E unlinkLast(Node<E>)   //删除最后一个元素
E unlink(Node<E>)   //删除节点x
Node<E> node(int)   //查找第index个元素
```

### linkFirst
将元素e链接到链表最前面，要检查头结点为null的情况。
```
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null) //如果原先链表为空，要给尾结点复制，头结点和尾结点都指向当前节点
            last = newNode;
        else    //将f的前指针指向新节点
            f.prev = newNode;
        size++;
        modCount++;
    }
```

### linkLast
将元素e链接到链表最后面，要检查尾结点为null的情况。
```
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null) //如果原先链表为空，要给头结点复制
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

### linkBefore
将元素e链接到succ节点之前，要注意succ就是头结点情况。
```
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```
### unlinkFirst
删除第一个元素，并返回第一个元素的值，注意处理链表只有一个元素情况，要将尾结点赋值为`null`。函数功能实际上是删除`f`以及所有`f`之前的元素，在调用时已经确定`f`是头结点。有3个方法调用了该函数，`removeFirst()`,`poll()`,`pollFirst()`。
```
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
### unlinkLast
与`unlinkFirst`相似。
```
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```
### unlink
在链表中删除节点`x`。
```
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```
### node
在链表中查找第`index`个元素。链表可以从两端遍历，首先判断从哪一端更近，然后开始遍历。
```
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
## List接口方法的实现
### get
调用`node(int index)`实现。

### set
调用`node（int index）`实现。

### add
调用`linkLast`或`linkBefore`实现。

### remove
调用`unlink`实现。允许`null`值得存储，如果是`null`值，调用`==`判断；如果不是`null`值，调用`equals`判断。

## Deque接口方法的实现
### addFirst, offerFirst, push
调用`linkFirst`实现。`offerFirst`, `push`调用`addFitst`。 `addFirst`调用`linkFirst`。

### addLast, OfferLast, add, offer
调用`linkLast`实现。`offer`调用`add`, `offerLast`调用`addLast`。`add`, `addLast`调用`linkLast`。

### removeFirst, remove, pollFirst, poll, pop
调用`unlinkFirst`实现。`remove`, `pop`调用`removeFirst`，`removeFirst`, `pollFirst`, `poll`调用`unlinkFirst`。

### removeLast, pollLast
调用`unlinkLast`实现。

### peek, peekFirst
直接查看头结点。

### peekLast
直接查看尾结点。

## 迭代器
只提供了`ListItr`。额外提供了倒序遍历的`DescendingIterator`。




