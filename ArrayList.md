# ArrayList
ArrayList实现了`List<E>, RandomAccess, Cloneable, java.io.Serializable`接口，除`List<E>`接口外都是标记接口，`RandomAccess`接口表示可以随机访问，`Cloneable`表示可以克隆，`Serializable`接口表示可以序列化。
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 几个关键属性
```
    //默认容量为10
    private static final int DEFAULT_CAPACITY = 10;
    //两个空数组，区别两种情况
    //初始化时，指定容量为0指定为该空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    //初始化时，调用默认构造函数时指定为该空数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    //保存所有元素的数组
    transient Object[] elementData; // non-private to simplify nested class access
    //当前元素个数
    private int size;
```
## 构造函数
默认构造函数`ArrayList()`，带初始容量的构造函数`ArrayList(int initialCapacity)`，使用一个集合来构造`ArrayList(Collection<? extends E> c)`。默认构造函数将`elementData`初始化为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，另外两个，当指定的容量不为0或者集合不为空，则会分配内存，如果指定的容量为0或者集合为空，则将`elementData`初始化为`EMPTY_ELEMENTDATA`，两者在扩容时有不同。
```
    //默认构造函数
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
```
    //指定初始化容量
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//如果指定容量为0
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
```
    //使用集合来构造
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // defend against c.toArray (incorrectly) not returning Object[]
            // (see e.g. https://bugs.openjdk.java.net/browse/JDK-6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {//如果集合为空
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## add
`add(E e)`调用了`add(E e, Object[] elementData, int s)`方法在数组最后插入，如果当前数组已经写满则扩容`grow()`。`add(int index, E e)`在`index`处插入元素，首先检查是否需要扩容，然后`index`之后的元素每个往后移动，最后将`e`放在`index`位置。
```
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size = s + 1;
    }
```
```
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }
```
```
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        System.arraycopy(elementData, index,
                         elementData, index + 1,
                         s - index);
        elementData[index] = element;
        size = s + 1;
    }
```
## get
```
    public E get(int index) {
        Objects.checkIndex(index, size);
        return elementData(index);
    }
```

## set
```
    public E set(int index, E element) {
        Objects.checkIndex(index, size);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
## remove
`remove`方法调用了`fastRemove`方法。
```
    public E remove(int index) {
        Objects.checkIndex(index, size);
        final Object[] es = elementData;

        @SuppressWarnings("unchecked") E oldValue = (E) es[index];
        fastRemove(es, index);

        return oldValue;
    }
```
```
    private void fastRemove(Object[] es, int i) {
        modCount++;
        final int newSize;
        if ((newSize = size - 1) > i)
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }    
```

## 扩容
