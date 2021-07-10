# HashSet

## 类层级关系

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

## 类相关属性

```java
// HashSet其实就是HashMap中的key，底层还是HashMap的数据结构
private transient HashMap<E,Object> map;
// 一个虚拟对象，用作存储Map的value值
private static final Object PRESENT = new Object();
```

## 构造函数

```java
// 无参构造，初始化HashMap容器
public HashSet() {
    map = new HashMap<>();
}
// 传入一个集合来初始化容器
public HashSet(Collection<? extends E> c) {
    // 1 这里假如我传入的集合大小是8，那么 8/.75f 取整之后的结果是11
    //   还达不到HashMap的默认初始容量，所以取16容量大小初始化
    // 2 集合大小是12，那么计算出来结果是19，则取19容量大小
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
// 通过指定初始化容量大小和加载因子来初始化容器
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
// 
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
// 采用有序的链表来初始化
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

## 特点

1. 因为HashSet的底层是HashMap，因此它的值是不会重复的，可以达到一个去重的效果。
2. 不能保证数据的存储顺序，即是无序的。
3. HashSet其实就是HashMap的键的集合。

## 常用方法底层原理

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

​		往HashSet中新增值其实就是往HashMap中put一个元素，key存储的是传入的值，value是HashSet中的一个成员属性PRESENT（虚拟对象）。

```java
// 删除一个元素，也是同样的操作，直接调用HashMap的remove方法
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

​		HashSet中是通过迭代器去遍历键的。

Thank ！