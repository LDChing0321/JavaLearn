# Vector

## 类层级关系

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 类相关属性

```java
// 容器本身
protected Object[] elementData;
// 容器大小，初始值为0，相当于size
protected int elementCount;
// 扩容的量
protected int capacityIncrement;
```

## 构造函数

```java
// 无参数构造，默认初始化容量大小为10，capacityIncrement默认为0
public Vector() {
    this(10);
}
// 指定容量初始化，capacityIncrement默认为0
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}
// 指定 初始化容量 和 扩容的量 来初始化容器
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}
// 传入一个集合来初始化容器
public Vector(Collection<? extends E> c) {
    elementData = c.toArray();
    elementCount = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
}
```

## 特点

1. 和ArrayList一样，底层是以动态数组来实现的。
2. 随机访问速度快，插入删除时效率低。
3. 线程安全。

## 常用方法

```java
public synchronized void addElement(E obj) {
    // 容器的操作次数
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = obj;
}
```

Vector自身拥有addElement方法，往容器新增元素。首先，它会做一个预判断，判断新增一个元素之后是不是超过了容器的当前容量。如果超过了，会进行一个扩容的操作。

```java
private void ensureCapacityHelper(int minCapacity) {
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

那它扩容是怎么做的呢？它首先会判断capacityIncrement（容量的增长系数）大于0的话，则扩容后的数组容量是**oldCapacity + capacityIncrement**；如果小于或等于0，则扩容 **2 * oldCapacity** (即原数组容量的两倍)，然后会对新的容量做一个边界值的处理，最后是对数组做一个拷贝迁移的动作。

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

如果不需要扩容，容量还没达到瓶颈，则直接往数组中添加新元素。

```java
elementData[elementCount++] = obj;
```

类似的，Vector还提供了另外一个新增元素的方法**insertElementAt**，它可以往指定的索引插入新元素。

```java
public synchronized void insertElementAt(E obj, int index) {
    modCount++;
    if (index > elementCount) {
        throw new ArrayIndexOutOfBoundsException(index
                                                 + " > " + elementCount);
    }
    ensureCapacityHelper(elementCount + 1);
    System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    elementData[index] = obj;
    elementCount++;
}
```