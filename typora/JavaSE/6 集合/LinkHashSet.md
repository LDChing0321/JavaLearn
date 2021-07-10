# LinkHashSet

## 类层级关系

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

## 构造函数

```java
// 调用父类HashSet的构造方法
public LinkedHashSet() {
    super(16, .75f, true);
}
// 其实底层就是LinkedHashMap
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

```java
public LinkedHashSet(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor, true);
}
public LinkedHashSet(int initialCapacity) {
    super(initialCapacity, .75f, true);
}
public LinkedHashSet(Collection<? extends E> c) {
    super(Math.max(2*c.size(), 11), .75f, true);
    addAll(c);
}
```

## 常用方法底层原理

```java
// 往HashMap中put一个元素，key为传入的值，value为HashSet的成员属性PRESENT（虚拟对象）
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

后面的方法参考LinkedHashMap

Thank ！