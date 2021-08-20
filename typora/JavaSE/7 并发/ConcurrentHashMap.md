# ConcurrentHashMap

## 类层级关系

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
    implements ConcurrentMap<K, V>, Serializable
```

```java
// 此表的默认初始容量，在构造函数中未指定时使用。
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 此表的默认加载因子，在构造函数中未指定时使用。
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 此表的默认并发级别，在构造函数中未指定时使用。
static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 最大容量，在两个带参数的构造函数隐式指定更高值时使用。 必须是 2 <= 1<<30 的幂，以确保条目可使用整数索引。
static final int MAXIMUM_CAPACITY = 1 << 30;
// 每段表的最小容量。 必须是 2 的幂，至少是 2 以避免在延迟构造后立即调整大小。
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
// 允许的最大段数； 用于绑定构造函数参数。 必须是小于 1 << 24 的 2 的幂。
static final int MAX_SEGMENTS = 1 << 16;
// 在求助于锁定之前，未同步重试 size 和 containsValue 方法的次数。 这用于避免在表进行连续修改时无限制的重试，这将导致无法获得准确的结果。
static final int RETRIES_BEFORE_LOCK = 2;
// 用于分段索引的掩码值。 密钥散列码的高位用于选择段。
final int segmentMask;
// 段内索引的移位值。
final int segmentShift;
// 段，每个段都是一个专门的哈希表。
final Segment<K,V>[] segments;
```

## 构造函数

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 计算并发级别ssize，保证并发级别是2的n次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 这里先采取默认的并发级别16，即sshift为4
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```