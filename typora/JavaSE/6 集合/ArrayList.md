# ArrayList

## 类的层级关系

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 类的相关属性

```java
// ArrayList的默认容量
private static final int DEFAULT_CAPACITY = 10;
// 空实例，用于共享数据
private static final Object[] EMPTY_ELEMENTDATA = {};
// 实际用来装载数据的容器
private transient Object[] elementData;
// 当前数组的元素大小
private int size;
```

## 构造函数

根据传入的容量大小初始化容器，容器大小不能小于0

```java
public ArrayList(int initialCapacity) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
}
```

根据默认容量10来初始化容器

```java
public ArrayList() {
    super();
    this.elementData = EMPTY_ELEMENTDATA;
}
```

将传入的集合赋值给elementData，size等于传入的集合实际大小

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
}
```

## 常用方法底层原理

​		每次增加数据的时候，需要判断是否需要扩容。如果当前容器为空的话，minCapacity取DEFAULT_CAPACITY和（size+1）的最大值。否则minCapacity取size+1。

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

如果minCapacity大于容量的话，则进行扩容。

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
// 扩容方法将扩容原来数组的1.5倍，这里扩容将数据迁移用到的方法是Arrays.copyOf
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

删除之前，会有一个数据越界的检查，如果index≥size的话，抛出数组越界的异常。

要删除节点的索引的后一位开始到末尾节点的索引都是需要向前移动一位的。

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

​	等讲到LinkedList的时候，会具体分析两者的特点以及效率问题	

Thank!

