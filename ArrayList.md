# ArrayList
ArrayList实现了`List<E>, RandomAccess, Cloneable, java.io.Serializable`接口，除`List<E>`接口外都是标记接口，`RandomAccess`接口表示可以随机访问，`Cloneable`表示可以克隆，`Serializable`接口表示可以序列化。
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 几个关键属性
```
    //默认容量
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
