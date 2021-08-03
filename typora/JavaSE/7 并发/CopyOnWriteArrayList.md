# CopyOnWriteArrayList

## 介绍 

​		我们平时开发用的比较多的是ArrayList，追溯源码知道它在多线程环境下是存在问题的，通过一个示例演示一下。

```java
public class ArrayListTest {

    public List<Integer> list = new ArrayList<>();

    @Test
    public void demo() throws InterruptedException {
        // 开启两条线程进行写操作
        for (int i = 0; i < 2; i ++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 1; i <= 10; i ++) {
                        list.add(i);
                    }
                }
            }).start();
        }
        Thread.sleep(1l);
        System.out.println(list);
    }
}
```

![](C:\Users\13509\Desktop\java_learn\JavaLearn\photo\JavaSE\集合\Concurrent\ArrayList并发存在的问题.png)

​		当检测到对象的并发修改时，单线程或并发场景下可能会出现ConcurrentModificationException，这是因为它采用了**fail-fast**机制（即快速失败机制）。		

​		它怎么检测出有其他线程对其进行修改呢？它通过**checkForComodification**这个方法去判断的，它里面有两个参数**modCount**、**expectedModCount**。第一个参数**modCount**肯定都知道很多容器都具有这个属性，它是用于记录容器的修改次数。而**expectedModCount**，在迭代遍历的预备阶段会将**modCount**赋值给**expectedModCount**，从而多个线程并发的修改将导致**modCount**的值产生变化，**checkForComodification**就会判断两个变量的值是否相等，这样就能检测出是否有其他线程进行修改操作（这里指**非迭代器本身的add或remove操作**，这里的修改也是指新增和删除操作），随后就会执行快速失败的机制，避免未来程序以各种不确定性的原因而引发异常。

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

​		这里有小伙伴就会问为什么迭代器本身的add或remove方法就不会触发**fail-fast**机制呢？这是因为迭代器的add和remove方法和对象本身的两个方法是不太一样的，迭代器每一次的新增和删除操作都将expectedModCount这个值刷新了，每次在迭代的时候**expectedModCount**和**modCount**都是相等的，所以不会触发**fail-fast**机制。

```java
// 迭代器的add方法
public void add(E e) {
    checkForComodification();

    try {
        int i = cursor;
        ArrayList.this.add(i, e);
        cursor = i + 1;
        lastRet = -1;
        // 每进行一次add操作就会刷新expectedModCount
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}

// 迭代器的remove方法
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        // 每进行一次remove操作就会刷新expectedModCount
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

​		ArrayList存在并发问题，而解决的办法可以使用**Vector**或者**SynchronizedList**。**Vector**主要是在方法声明处添加了**synchronized关键字**使其变为安全的容器；**SynchronizedList**则在方法的内部代码块使用**synchronized关键字。**

​		这两者的效率挺慢的，不推荐使用。

​		CopyOnWriteArrayList就登场了！

​		COW涵盖**写时复制、读写分离**的思想在里面。**写**包括了新增、修改和删除操作，**复制**意为执行这些操作的时候克隆源数组并且分配一块新的内存用来保存。在并发场景下，**读线程**和**写线程**之间互不影响。**读线程**读的是源数组的内存空间；而**写线程**操作的是新克隆出来的内存空间。

​		CopyOnWriteArrayList是怎么保证线程安全的？进行写操作的时候，在方法内部加入**可重入锁ReentrantLock**。可重入锁的意思是一个线程可多次获得同一把锁，多次获得就要多次释放，否则锁可能不会释放。它与synchronized都是可重入锁，前者需要手动加锁和释放锁，使用起来更灵活；后者则是自动加锁和释放锁。ReentrantLock还支持公平锁和非公平锁，这个先预热一下，后续会详细介绍。

## 源码解读

### 类层级关系

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

### 成员属性

```java
// 可重入锁
final transient ReentrantLock lock = new ReentrantLock();
// 用volatile标记，并发下实现内存可见性。
// 实际存储数据的数组，外界只能通过getArray/setArray的方法访问。
private transient volatile Object[] array;
```

### 构造函数

```java
// 无参构造
public CopyOnWriteArrayList() {
    // 初始化array
    setArray(new Object[0]);
}
// 接收一个集合作为参数的构造函数
public CopyOnWriteArrayList(Collection<? extends E> c) {
    // 临时变量
    Object[] elements;
    // 判断集合c是不是CopyOnWriteArrayList
    if (c.getClass() == CopyOnWriteArrayList.class)
        // 将集合c的array赋值给临时变量
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        // 将集合c转化为数组
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            // 将数组拷贝之后赋值给临时变量
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    // 设置初始化状态的array
    setArray(elements);
}

// 接收一个数组作为参数的构造函数
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

### 成员方法

​		add实现原理和ArrayList差不多，只是增加了锁的机制。

```java
// 将指定的元素附加到此列表的末尾
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 将源数组克隆一份形成新数组，这里可以体现“写时复制”
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 末尾添加元素
        newElements[len] = e;
        // 将新数组赋值给array
        setArray(newElements);
        return true;
    } finally {
       	// 释放锁
        lock.unlock();
    }
}
```

​		重点看看remove方法

```java
public boolean remove(Object o) {
    // 将array赋值给snapshot
    Object[] snapshot = getArray();
  	// 计算对象o 在array数组的索引位置
    int index = indexOf(o, snapshot, 0, snapshot.length);
   	// 索引有效则删除对象o
    return (index < 0) ? false : remove(o, snapshot, index);
}
// 计算要删除对象的索引下标
private static int indexOf(Object o, Object[] elements,
                           int index, int fence) {
    if (o == null) {
        for (int i = index; i < fence; i++)
            if (elements[i] == null)
                return i;
    } else {
        for (int i = index; i < fence; i++)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}


private boolean remove(Object o, Object[] snapshot, int index) {
    // 这里要考虑多线程的情况，进入该方法之前snapshot可能已经过时
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // array数组赋值给current变量
        Object[] current = getArray();
        int len = current.length;
        // 当snapshot和current内存不同时，考虑时多线程情况下snapshot被修改了
        if (snapshot != current) findIndex: {
            // 取index和len的最小值
            // 重新获取新的数组大小
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                // 遍历找到索引下标
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len);
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```