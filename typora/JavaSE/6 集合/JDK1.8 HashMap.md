# JDK1.8 HashMap





## 前言

相信看过HashMap1.7的朋友应该知道，1.7底层的设计是动态数组+链表的结构，而1.8引入了红黑树的概念。为什么要引入红黑树？

## HashMap的前世今生

我们都知道1.7版本使用单链表解决哈希冲突的问题。单链表，即链表中每个节点都有一个next指针，指向下一个节点，这样就形成单向的一个链表。试想，设这个链表的长度达到n，每当读取该链表中的其中一个节点时，就要从链表的首节点开始自旋，直到返回正确的节点，时间复杂度是O(n)。

而1.8后，采用双向链表+红黑树的方式，其实并不是一开始就使用双向链表，而是有个过渡期，中间会将单链表转换成双向链表，再由双向链表转成红黑树。双向链表，即有两个指针分别指向前后对应的节点。而红黑树是一种自平衡二叉查找树，它结构看起来简单、清晰，它实现起来算是比较复杂的😅，但是它有着比较可观的效率，可以在O(log n)的时间内完成增、删、查。

## 类层级关系

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

## 成员属性

相比1.7版本多了**TREEIFY_THRESHOLD**、**UNTREEIFY_THRESHOLD**、**MIN_TREEIFY_CAPACITY**这几个属性，也是红黑树的几个关键属性。

```java
// 默认初始容量 - 必须是 2 的幂。
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量, 必须是 2 的幂 而且要小于 1<<30。
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认负载因子。
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 链表长度达到8后转化为红黑树。该值必须大于 2 且至少应为 8
static final int TREEIFY_THRESHOLD = 8;
// 链表长度收缩到6后退化为链表。
static final int UNTREEIFY_THRESHOLD = 6;
// 红黑树的最小容量，哈希表要达到这个阈值才会转成红黑树
static final int MIN_TREEIFY_CAPACITY = 64;
// 哈希表，在第一次使用时初始化，并根据需要调整大小。 分配时，长度始终是 2 的幂。 
transient Node<K,V>[] table;
transient Set<Map.Entry<K,V>> entrySet;
// 此映射中包含的键值映射数。
transient int size;
// 该 HashMap 被结构修改的次数
transient int modCount;
// 这里有个细节，首次初始化或者容量小于16时，threshold是通过【容量 * 负载因子】计算的。
// 后面扩容的时候，容量大于等于16时，则扩容后的threshold是 旧阈值的两倍，这是为什么？
// 因为容量16之前【旧阈值 的 两倍】和 【容量 * 负载因子】 可能会有不相等的情况，直到容量大于等于16，它们两者计算结果相等。
// 【一个数的两倍 = 这个数左移一位】比实际乘法运算效率要来得快些。
int threshold;
// 哈希表的负载因子。
final float loadFactor;
```

## 关键属性

**DEFAULT_LOAD_FACTOR**（加载因子）：HashMap为什么默认加载因子选择0.75？作者给出答案是它需要衡量时间和空间的成本，选取最折中的一个值作为因子。假设哈希表长度为16，如果因子过高，比如1，则哈希表节点长度达到16才扩容，虽然可以节省空间成本（扩容频率少，扩容后容量是原本的两倍），提高了空间利用率，但同时增加查询节点的时间成本，冲突的机会也因此加大；因子过低，比如0.5，则哈希表节点长度达到8开始扩容，虽然可以节省查询成本，但是空间利用率相对来说并不是最优的，所以这里作者也是选取了0.75作为因子，这个值并不是固定的，开发者使用时可以自行修改。

**DEFAULT_INITIAL_CAPACITY**（默认初始 容量1<<4）：为什么要保证容量是2的次幂？首先换想一下，作者为什么使用 （n-1）& hash 的这种方式计算索引位置（**n是哈希表长度**）？hash值这里暂且理解成一个“**特征值**”，我们都知道两个数做 & 运算，最后结果不会超过较小的数，所以（n - 1）起到限制的作用，**这样能够保证计算出来的索引位不会超出哈希表的长度**。**且当n是2的次幂时 （n-1）& hash  与  n % hash式子算出的结果相同，为什么不用 n % hash 进行计算，是因为 & 比 % 效率略高一点**。![](../../../photo/JavaSE/集合/HashMap/&和%的计算耗时.png)

## 构造函数

```java
// 构造一个具有指定初始容量和负载因子的空HashMap 。
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    // 初始化扩容阈值
    this.threshold = tableSizeFor(initialCapacity);
}
// 构造一个具有指定初始容量和默认负载因子 (0.75) 的空HashMap 。
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 构造一个具有默认初始容量 (16) 和默认负载因子 (0.75) 的空HashMap 
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 使用与指定Map相同的映射构造一个新的HashMap 。 HashMap是使用默认负载因子 (0.75) 创建的，初始容量足以在指定的Map 中保存映射。
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
// 返回首个大于或等于给定目标容量的二次幂数。
// 假设给定7，则返回8。
// 给定17，则返回32。
// 给定16，则返回16。
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

**tableSizeFor**这个方法简单介绍一下，认真理解之后其实没那么难理解😏，这里就举个简单的例子吧。

1. 首先给定cap = 9，9的二进制是 **1001**【前面省略28个零，总共32位】。
2. 临时变量 n = cap - 1 = 8，二进制是 **1000**。
3. 【n】和【n 无符号右移 1位 】两个做或运算，结果赋予n。
4. 重复第3步，不同的是当前右移的位数是上一次位数的两倍，结果继续赋予n。**n其实就是记录上一次运算的结果，等下一次运算时就用n作为基础的数值。**

$$
①n = 1000|(1000 >>> 1)=1000|0100=1100
$$

$$
②n = 1100|(1100 >>> 2)=1100|0110=1110
$$

$$
③n = 1110|(1110 >>> 4)=1110|0111=1111
$$

计算到这一步再往下面计算结果都是**1111**，最后一步还要将结果n加1得到16，所以传入9函数将返回16。

也可以这样理解：给定一个容量大小，将容量减去1得到一个二进制数，从这个二进制数最高位是1的位置开始，向最低位依次填充1。例如容量19，二进制是**10011**，减1之后是**10010**，依次填充1之后得到**11111**，然后再加上1，最终函数返回**32**。
		总结一句话：**返回首个大于或等于给定目标容量的二次幂数**。

## 插入元素过程

我们先从最常见的put方法开始吧，当我们put一个元素的时候，首先它会根据传入的key去做hash运算，hash运算步骤如下：

1. 取key的hashCode值（根据key的数据类型调用对应的hashCode方法，此处以String类型为例）。
2. 将hashCode值进行无符号右移16位。
3. 将 原hashCode值 与 右移之后的值 进行异或运算得到hash值（也就是第一步和第二步做异或运算）。

**为什么要右移16位？**

**主要是为了减少碰撞，更好的避免下标冲突**。这样设计主要原因是能够保留hashcode的**高16位**与**低16位**各自的特征。源码是通过**(n-1)&hash** 去计算索引下标的，有没有想过，如果**n-1**的值还不到16位呢？那是不是hash的高16位是没有参与运算的？例如n-1=2，2的二进制是0010，假设hash值是32并且是原始的hashcode（没有做高低位的运算），直接与0010做&运算，那么hash只有低16位参与了其中，高16位被忽略掉了，这样不利于计算下标时的均匀分布，作者想通过融合hash高低位的二进制特征，最后使得索引能够均匀分布。

看过1.7版本的小伙伴应该知道，1.8版本的hash算法比1.7版本的逻辑稍微简单点，是因为作者已经引入了红黑树去解决冲突问题了，考虑到前后效率平衡的问题，所以也就没必要去增加hash逻辑的复杂度了。

![](../../../photo/JavaSE/集合/HashMap/1.8版本hash算法.png)

上面英文简述部分的大致意思大概就是上述所描述的那样了，不过这仅限于本人所思考的哦，不知道小伙伴们有没有其他不同看法😃

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 此为String类的hashCode方法
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            // 一个数迭代运算的过程
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
} 
```

我们具体来看看put的步骤

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果tab为空，则初始化哈希表
    if ((tab = table) == null || (n = tab.length) == 0)
        // 用n记录resize后哈希表的大小
        n = (tab = resize()).length;
    // 根据哈希表的长度-1 与 hash值 做 与运算
    // 例如哈希表长度为32，则计算 31 & hash
    // 如果该索引位上为空，才可赋值（在该索引位上新建一个节点）
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 如果该索引位上不为空，说明该位置上 已经存在一个节点或者一条链表或者一颗红黑树
        // 假如表中存在相同的key，则用e记录下来
        Node<K,V> e; K k;
        // ① 该索引位上的hash值与新计算出来的hash值相等
        // ② 该索引位上的key与新的key相等
        // 满足①和②则进行覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果索引位上的元素类型是TreeNode，即此时链表已经是一颗红黑树了。
        else if (p instanceof TreeNode)
            // 使用红黑树的方式插入元素
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 开始遍历链表
          	// 此时链表的长度是小于8的，还没有转化为红黑树
            for (int binCount = 0; ; ++binCount) {
                // 如果e是空的话，说明链表的当前元素是最后一个元素
                // 遍历到链表的最后都没有匹配到相同的key，则在链表的末尾新增一个元素
                if ((e = p.next) == null) {
                    // 如果p.next为空，则根据key、value初始化一个新的元素赋值给p.next
                    // 这里可以看出采用的是尾插法
                    p.next = newNode(hash, key, value, null);
                    // 链表增加了元素，需判断链表的长度是否达到了8
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 达到了8则转化为红黑树
                        treeifyBin(tab, hash);
                    // 跳出循环，此时的e为空
                    break;
                }
                // e不为空的情况
                // 即遍历过程中，发现链表中存在相同的key
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) // **
                    // 跳出循环，执行（existing mapping for key）
                    break;
                // 指向下一个节点，继续遍历
                p = e;
            }
        }
        // 覆盖哈希表中已经存在的key，即上面标记为**处
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 记录操作次数
    ++modCount;
    // 新增元素之后如果超过扩容阈值，需要进行扩容。
    // 注意1.7是先扩容再插入元素，而1.8是先插入元素再扩容
    if (++size > threshold)
        resize(); 
    afterNodeInsertion(evict);
    return null;
}
```

如果源码和注释看得不够清晰的，还可以看下具体的流程图哦

![](../../../photo/JavaSE/集合/HashMap/put过程1.8.drawio.png)

经过上面put的源码解析，可以简单总结1.7和1.8的不同之处：

 1）**1.8**的底层数据结构是**数组+链表+红黑树**，链表长度达到8的时候会将链表转化为红黑树；1.7是数组+链表。

 2）**1.8**是先插入元素再扩容，**1.7**是先扩容再插入元素。

 3）**1.8**采用链表尾插法，**1.7**采用链表头插法。

 4）hash运算不同。1.7使用了4次位运算和5次异或；1.8只用了2次扰动处理：1次位运算+1次异或。可以看出1.7更注重在hash算法层面去分配索引位，1.8则是增加红黑树结构去解决冲突问题。时间复杂度从O（n）变成O（log n）从而提高了效率。

如果当我们新增一个节点，计算出这个节点的索引位上已经有一颗是红黑树了，这个时候就不能再使用链表尾插了，要根据红黑树的方式插入节点。

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    // 从当前索引位的红黑树的根节点开始遍历
    TreeNode<K,V> root = (parent != null) ? root() : this;
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // 当前这个树节点的hash 大于 新节点的hash
        if ((ph = p.hash) > h)
            // 此时，新节点应该插入的方向是左边
            dir = -1;
        // 当前这个树节点的hash 小于 新节点的hash
        else if (ph < h)
            // 此时，新节点插入的方向是右边
            dir = 1;
        // 新节点与此时树节点 的key相同，覆盖其值
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            // 直接返回当前的这个树节点
            return p;
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {

            //判断新节点没有实现Comparable接口 或者
            //k和pk的compareTo等于0(即新节点和原本树节点的key通过compareTo比较后相等)
            
          	// 这里主要是搜寻树结构中是否有equals方法比较后相同的节点
		   // 找到之后返回出去
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                // 从树的根节点沿着左侧开始搜寻，左侧如果找到立即返回，不再搜寻右侧树
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            // 走到这里，说明上一步没有search到相同的节点。 
            // 这个方法是最终确认要插入的节点是位于树的左侧还是右侧，
            // 调用JNI接口生成hashcode再次比较，最终确认插入的方向。
            dir = tieBreakOrder(k, pk);
        }
	    // 走到这一步， 插入的方向已经确定了。
        TreeNode<K,V> xp = p;
     	// 方向确定之后， 判断左子节点或右子节点是否为空， 不为空则需要进行下一次的遍历
        // 为空，则进行插入的逻辑
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            // 当前树节点的next节点
            Node<K,V> xpn = xp.next;
            // 新建一个树节点，并设置新节点的next节点为 xpn节点
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                // 置于左侧
                xp.left = x;
            else
                // 置于右侧
                xp.right = x;
            // 当前树节点的next节点 重新指定为 此时的新节点
            xp.next = x;
            // 新节点的parent、prev均指向当前树节点（xp）
            x.parent = x.prev = xp;
            if (xpn != null)
                 // 当前树节点的next节点的prev指向当前新节点
                ((TreeNode<K,V>)xpn).prev = x;
            // 维护红黑树平衡
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

这个方法关键是确认插入树的方向是左侧还是右侧，并且是在树的尾部进行插入，插入之后要维护红黑树的平衡，以下是流程图：

![](../../../photo/JavaSE/集合/HashMap/以树的方式插入新节点.png)

从单链表转为双链表，再从双链表转为红黑树主要是这个方法**treeifyBin**，它的实现逻辑是怎样的呢？ 首先转成双链表。

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 当哈希表长度小于 MIN_TREEIFY_CAPACITY时，先利用扩容的方式去代替红黑树，可以理解还没达到转化红黑树的条件
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 根据哈希表的长度和hash进行与运算后，得到链表的头节点
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 当哈希数组长度大于MIN_TREEIFY_CAPACITY时，这时会转换为红黑树，中间的过程也不是一步到位的，而是先将哈希表的当前索引位上的单链表转成双向链表，再由双向链表转成红黑树。
        
        // 首先，定义头尾节点，主要做辅助作用。
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 遍历单链表，每遍历一个链表节点就生成一个树节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // 如果尾节点为空，说明没有根节点，第一次循环肯定为空
            if (tl == null)
                // 头节点指向当前节点，这里是双链表的初始化工作，
                // 先处理 头节点开始
                hd = p;
            else {
                // 这里说明尾节点不为空，因为已经做了初始化工作了
                // 将当前遍历的链表节点的prev指向尾节点
                p.prev = tl;
                // 尾节点的next指向当前节点
                tl.next = p;
            }
            // 此时，尾节点指向当前节点，这里形成了一个闭环，因为是双链表
            tl = p;
        } while ((e = e.next) != null); // 指向下一个单链表节点，继续遍历链表
        
        // 上面的步骤已经把单向链表转换为双向链表了
        if ((tab[index] = hd) != null)
            // 将双向链表树化
            hd.treeify(tab);
    }
}
```

​	这里是双链表转红黑树哦

```java
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    // 此时的x是双向链表的头节点，从头开始遍历。
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        // x的next节点
        next = (TreeNode<K,V>)x.next;
        // 定义左右子节点
        x.left = x.right = null;
        // 初始化树的根节点为x
        if (root == null) {
            // 父节点为空
            x.parent = null;
            // 根节点为黑色
            x.red = false;
            // x赋值给root节点
            root = x;
            // 进行下一次循环，此时x为x的next节点
        }
        else 
            // 根节点已存在
            // 记录当前节点的key和hash值
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            // 定位当前节点需要插入到树的什么位置
            // 遍历树结构，从根节点开始
            for (TreeNode<K,V> p = root;;) {
                // ph -> 当前树节点的hash值
                // dir -> 标记方向，左或右 
                int dir, ph;
                // 记录当前树节点的key
                K pk = p.key;
                // 如果当前树节点的hash值 大于 当前链表元素的hash值
                if ((ph = p.hash) > h)
                    // 标记当前树节点为-1，即左侧
                    dir = -1;
                else if (ph < h)
                    // 否则，为右侧
                    dir = 1;
               	// 走到这步，说明此时树节点和新节点的key 的内存地址以及两者的equals均不相等
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 这个方法是最终确认要插入的节点是位于树的左侧还是右侧，
           		   	// 调用本地方法生成hashcode再比较。
                    dir = tieBreakOrder(k, pk);
				
                // 判断是否已经遍历到叶子节点了，因为最后是插入到树的尾部
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 插入后维护树平衡
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        // 继续下一次循环
        }
    }
	// 用来将root节点放入到哈希槽中，保证其处于哈希桶的头部
    moveRootToFront(tab, root);
}
```

​	这里简述一下就是，将双链表的节点一个一个地插入到树中，中间需要判断hash值的大小知道插入的方向是在左侧还是右侧，根据方向然后插入到树的叶子节点处。

![](../../../photo/JavaSE/集合/HashMap/将双链表树化的过程.png)

​	下面的重头戏是树如何保持平衡？首先了解一下红黑树的一些特点，它是一种**自平衡二叉查找树**，主要有以下性质：

1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子节点都是黑色。
4. 每个红色节点必须有两个黑色的子节点（或者说红色节点的父节点和子节点均是黑色的）。
5. 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点。

![](../../../photo/JavaSE/集合/HashMap/Red-black_tree.png)

​	根据上面图可以看到，所有左子节点比它父节点都小，所有右子节点比它父节点都大。
​			也正是因为上面的性质，才体现红黑树自平衡的特质，不会像普通二叉查找树一样出现瘸子的现象，**也同时保证了树从根到叶子结点最长路径不会超过最短路径的2倍**。

​	下面介绍维护树的平衡就是要保证上面红黑树的性质，在此之前说说保持平衡的几种情况（暂且只讨论平衡的情况，颜色到后面再讨论）：

​	① 失衡情况 - 右右：**需要以节点8进行左旋**

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右右1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右右2.png)

② 失衡情况 - 左左：**需要以节点4进行右旋**

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左左1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左左2.png)

③ 失衡情况 - 左右：

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左右1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左右2.png)

④ 失衡情况 - 右左：

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右左1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右左2.png)

以上都是**新元素**的父亲没有兄弟元素的情况下进行的平衡，如果有一个兄弟元素并且是红色，这时候又要分不同情况了：

①右右情况-父亲元素有一个红色兄弟元素：

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右右伴有兄弟元素1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右右伴有兄弟元素2.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-右右伴有兄弟元素3.png)

①左左情况-父亲元素有一个红色兄弟元素：

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左左伴有兄弟元素1.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左左伴有兄弟元素2.png)

![](../../../photo/JavaSE/集合/HashMap/红黑树失衡情况-左左伴有兄弟元素3.png)

上面所有的情况基本上就是下面代码所有表达的内容了。

```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    // 默认插入的元素是红色
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        // 初始化根节点
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        // 父亲元素是黑色，没有破坏平衡
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        // 父亲是爷爷的左子元素
        if (xp == (xppl = xpp.left)) {
            // 父亲有一个红色的兄弟元素
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                // 如果这里不成立，那么就是左左情况，直接右旋即可。
                if (x == xp.right) {
                    // 成立的话就是左右情况，先左旋然后再右旋
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        // 右旋
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        // 父亲是爷爷的右子元素
        else {
            // 父亲有一个红色的兄弟元素
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                // 如果这里不成立，那么就是右右情况，直接左旋即可。
                if (x == xp.left) {
                    // 成立的话就是右左情况，先右旋然后再左旋
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        // 左旋
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

插入完元素之后还要判断是否要进行扩容

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
    	// 旧容量>0
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 新容量是原来的两倍，并没有超过MAXIMUM_CAPACITY
            // 并且旧容量要>=16
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 新的扩容阈值也是原来阈值的两倍
                newThr = oldThr << 1; // double threshold
        }
    	// 旧阈值大于0
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 新容量等于旧阈值
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 采用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	// 新阈值等于0
        if (newThr == 0) {
            // 计算新阈值 -> 新容量 * 加载因子
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 遍历旧的哈希表
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 表的当前槽位不为空
                if ((e = oldTab[j]) != null) {
                    // 清空当前槽位，原来槽位的数据已经换成e来记录了，相当于一个临时变量
                    oldTab[j] = null;
                    // 当前槽位只有一个元素
                    if (e.next == null)
                        // 根据当前元素的hash 和 新哈希表的槽位数 做 & 运算，并将当前元素存储到新哈希表对应的索引位置
                        newTab[e.hash & (newCap - 1)] = e;
                    // 当前槽位是一颗红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 当前槽位是一个链表
                    else { // preserve order
                        // 定义lo的头元素和尾元素
                        Node<K,V> loHead = null, loTail = null;
                        // 定义hi的头元素和尾元素
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 遍历链表
                        do {
                            next = e.next;
                            // 当前槽位的当前元素的hash 和 旧容量 做&运算等于0
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    // 初始化lohead元素
                                    loHead = e;
                                else
                                    // loTail的下一个元素指向当前元素
                                    loTail.next = e;
                                // 尾元素自然也是当前元素
                                loTail = e;
                            }
                            // 不等于0，和上面是一样的，只是不同变量存储而已
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 通过上面可以知道将原来链表进行分组
						// 做&运算等于0的元素分成一条链表
                        // 不等于0的分成另外一条链表
                        if (loTail != null) {
                            loTail.next = null;
                            // 做&运算等于0的这条链表放在新哈希表的j索引位上，也就是在原来的索引位上。
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 做&运算不等于0的这条链表放在新哈希表的j + oldCap 索引位上。
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```



### remove

```java
public V remove(Object key) {
    Node<K,V> e;
    // 删除之前先计算key的hash值
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    // tab -> 当前的数组
    // p -> 存放该key在数组的索引位上的链表
    // n -> 记录数组的大小
    // index -> 索引下标
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 此时p指向的是对应索引位上的链表
        (p = tab[index = (n - 1) & hash]) != null) {
        // node -> 要删除的元素
        // k -> 作为一个临时变量去存储元素的key
        // v -> 记录要删除元素的value
        Node<K,V> node = null, e; K k; V v;
        // ① 链表首元素的hash和形参的hash相同
        // ② 链表首元素的key相同
        // 同时满足①、②，则链表的首元素就是我们要删除的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 首元素的next元素不为空
        else if ((e = p.next) != null) {
            // 如果此时的p（即首元素）属于TreeNode类型，即此时的链表是红黑树的结构
            if (p instanceof TreeNode)
                // 采用红黑树的方式来获取要删除的元素
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 如果不是红黑树的结构，只是单纯的双向链表结构。
                // 则正常的遍历链表来找到要删除的元素
                // 以下do-while循环就是遍历链表去定位要删除的元素
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
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 如果要删除的元素属于TreeNode类型
            if (node instanceof TreeNode)
                // 则使用树结构的方式去删除该元素
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 如果要删除的元素刚好是链表的头元素
                // 直接将该数组索引位上的链表往下挪一个元素
                // 即断开头元素的引用
                tab[index] = node.next;
            else
                // 以上两种情况都不是，即说明此时要删除的元素处于链表的非头元素位置上。
                // 这种情况直接将头元素的next指针指向要删除的元素的next域即可。
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}

// 关键还是要看 要删除的元素处于红黑树 的结构状态下，该如何删除
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                          boolean movable) {
    // 定义n记录数组的长度
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    // 计算要删除的元素在数组中的索引
    int index = (n - 1) & hash;
    // first -> 获取对应索引的链表
    // root -> 指向first，起到一个游标的作用
    // rl -> 临时变量
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    // succ -> 要删除元素的下一个元素
    // pred -> 要删除元素的上一个元素
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    // pred为空说明要删除的元素是链表的头元素
    if (pred == null)
        // 将succ赋值给此时的索引位和first
        tab[index] = first = succ;
    else
        // 不为空则说明是非头元素
        // 将上一个元素的next指向succ
        pred.next = succ;
    // 如果succ不空，succ的上一个元素指向pred
    if (succ != null)
        succ.prev = pred;
    // 
    if (first == null)
        return;
    // 定位链表的根节点
    if (root.parent != null)
        root = root.root();
    // 从红黑树退化成链表
    if (root == null
        || (movable
            && (root.right == null
                || (rl = root.left) == null
                || rl.left == null))) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    // 还没达到退化链表的条件
    // p -> 待删除元素
    // pl -> 待删除元素的左子元素
    // pr -> 待删除元素的右子元素
    // replacement -> 替换元素
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    // 待删除元素有左右子元素
    if (pl != null && pr != null) {
        TreeNode<K,V> s = pr, sl;
        // 找到后继元素
        while ((sl = s.left) != null) // find successor
            s = sl;
        // c -> 后继元素的颜色
        // 交换后继节点和删除节点的颜色，最终的删除是后继节点，故平衡是否是以后继节点的颜色来判断的
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
        // sr -> 后继元素的右子元素（后继元素肯定不存在左子元素）
        TreeNode<K,V> sr = s.right;
        // pp -> p的父元素，待删除元素的父元素
        TreeNode<K,V> pp = p.parent; 
        // 如果后继节点与待删除元素的右孩子相等，类似于待删除元素有一个右子元素，右子元素没有任何子元素
        if (s == pr) { // p was s's direct parent
            // 待删除元素的父元素和右子元素位置互换
            p.parent = s;
            s.right = p;
        }
        else {
            // 如果待删除元素的右子元素有至少一个左子元素
            TreeNode<K,V> sp = s.parent;
            // 将后继元素的父元素赋值给p.parent，并判断不为空
            if ((p.parent = sp) != null) {
                // 后继元素是它父元素的左子元素
                if (s == sp.left)
                    // 修改父元素的左子元素为p
                    sp.left = p;
                else // 后继元素是它父元素的右子元素
                    // 修改父元素的右子元素为p
                    sp.right = p;
            }
            // 修改后继节点的右孩子值，如果不为null，同时指定其父节点的值
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        // 当前节点现在变成后继节点了，故其左孩子为null.
        p.left = null;
        // 修改当前节点的右孩子值，如果其不为空，同时修改其父节点指向当前节点
        if ((p.right = sr) != null)
            sr.parent = p;
        // 修改后继节点的左孩子值，如果其不为空，同时修改其父节点指向后继节点
        if ((s.left = pl) != null)
            pl.parent = s;
        // 修改后继节点的父节点值，如果其为null，说明后继节点现在变成了root节点
        if ((s.parent = pp) == null)
            root = s;
         // 当前节点是其父节点的左孩子
        else if (p == pp.left)
            pp.left = s;
        // 当前节点是其父节点的右孩子
        else
            pp.right = s;
        /**
        * sr-后继节点的右孩子节点（有一个孩子节点），
        * 如果右孩子节点不为空，删除节点后，替代节点就是其右孩子节点
        * 如果为空，那么替代节点就是其本身
        */
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    }
    // 删除节点有一个左子节点，左子节点作为替代节点
    else if (pl != null)
        replacement = pl;
    // 删除节点有一个右子节点，右子节点作为替代节点
    else if (pr != null)
        replacement = pr;
    // 删除节点没有子节点，直接删除当前节点
    else
        replacement = p;
    /**
    * 如果删除节点存在两个孩子节点，最终与后继节点交换后，删除的节点的位置位于后继节点的位置，那么此时删除节点所处的位置演变成：
    * a、只有一个孩子节点：(replacement = p.right) != p
    * b、没有孩子节点：replacement == p
    * 只有当删除节点与替换节点不相等的时候，才对删除节点进行删除操作
    */
    if (replacement != p) {
        // 从红黑树中将待删除节点（即当前节点移除）
        TreeNode<K,V> pp = replacement.parent = p.parent;
         // 是否为根节点
        if (pp == null)
            root = replacement;
         // 其父节点的左子节点
        else if (p == pp.left)
            pp.left = replacement;
        // 其父节点的右子节点
        else
            pp.right = replacement;
        // 节点的指向全部置NULL
        p.left = p.right = p.parent = null;
    }
        /**
        * 如果删除节点的颜色是红色，不会影响整棵树的黑色高度，毋需自平衡，根节点不会变化，如果是黑色，则需要进行自平衡，重新获取根节点
        * 注意：
        * 自平衡的时候 替代节点可能与删除节点相等：replacement == p
        * 自平衡的时候 替代节点可能与删除节点不相等：replacement ！= p
        */
    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);
        /**
        * 当 replacement == p 时，是先进行了红黑树的进行了平衡操作，再将这个节点从红黑树中移除
        * 这个地方我也没明白原理是什么，但是我按照这个步骤去走了一遍，确实这样操作来完成平衡，如果有哪位大神明白的，麻烦指导一下，谢谢！
        */
    if (replacement == p) {  // detach
         // pp-存储当前节点的父节点值
        TreeNode<K,V> pp = p.parent;
        // 当前节点的父节点指向NULL
        p.parent = null;
        // 如果父节点不为空，根据当前节点位于父节点的不同子节点，修改父节点的孩子节点值
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    // movable为true，需要将根节点移动到头节点，即数组所以位置指向的节点
    if (movable)
        moveRootToFront(tab, r);
}
```

```java
/**
* 红黑树删除节点后，平衡红黑树的方法
*
* @param root 根节点
* @param x    节点删除后，替代其位置的节点，这个节点可能是一个节点，也可能是一棵平衡的红黑树，在此处就当作一个节点，在该节点以上部分需要自平衡
* @return 返回新的根节点
*/
static <K, V> HashMap.TreeNode<K, V> balanceDeletion(HashMap.TreeNode<K, V> root, HashMap.TreeNode<K, V> x) {
    /**
    * 进入这个方法，说明被替代的节点之前是黑色的，如果是红色的不会影响黑色高度，黑色的会影响以其作为根节点子树的黑色高度
    * xp-父节点,xpl-父节点的左孩子,xpr-父节点的右孩子节点
    * 注意：
    *    进入该方法的时候 替代节点可能与删除节点相等：x == replacement == p
    *                  替代节点可能与删除节点不相等：x == replacement ！= p
    */
    for (HashMap.TreeNode<K, V> xp, xpl, xpr; ; ) {
        /**
        * 1、x == null，当 replacement == p 时，删除节点不存在，返回；
        *      因为当 replacement ！= p 时，replacement 肯定不会为null.在移除节点的方法中有三个地方对 replacement 进行赋值。
        *          1、if (sr != null) replacement = sr;
        *          2、if (pl != null) replacement = pl;
        *          3、if (pr != null) replacement = pr;
        * 2、x == root，如果替代完成后，该节点就是整棵红黑树的根节点，本身就是平衡的，直接返回
        */
        if (x == null || x == root)
            return root;
        else if ((xp = x.parent) == null) {
            // 如果父节点为空，说明当前节点就是根节点，设置根节点的颜色为黑色，返回
            x.red = false;
            return x;
        } else if (x.red) {
            /**
            * 被替换节点（删除节点）的颜色是黑色的，删除之后黑色高度减1，如果替换节点是红色，将其设置为黑色，可以保证
            * 1、与替换之前的黑色高度相等
            * 2、满足红黑树的所有特性
            * 达到平衡返回
            */
            x.red = false;
            return root;
            /**
            * 如果替换节点是黑色的，替换之前的节点也是黑色的，替换之后，以替换节点作为根节点子树黑色高度减少1，需要进行相关的自平衡操作
            * 1、替换节点是父节点的左孩子
            */
        } else if ((xpl = xp.left) == x) {
            /**
            * 情况1、父节点的右孩子（兄弟节点）存在且为红色
            * 处理方式：兄弟节点变黑，父节点变红，以父节点为支点进行左旋，重新获取兄弟节点，继续参与自平衡
            */
            if ((xpr = xp.right) != null && xpr.red) {
                xpr.red = false;
                xp.red = true;
                root = rotateLeft(root, xp);
                xpr = (xp = x.parent) == null ? null : xp.right;
            }

            // 不存在兄弟节点，x指向父节点，向上调整
            if (xpr == null)
                x = xp;
            else {
                // sl-兄弟节点的左孩子，sr-兄弟节点的右孩子
                HashMap.TreeNode<K, V> sl = xpr.left, sr = xpr.right;
                /**
                * 情况2-1：兄弟节点存在，且两个孩子的颜色均为黑色
                * 1、sr == null || !sr.red：兄弟的右孩子为黑色（空节点的颜色其实也是黑色）
                * 2、sl == null || !sl.red：兄弟的左孩子为黑色（空节点的颜色其实也是黑色）
                * 处理方式：兄弟节点为红色，替换节点指向父节点，继续参与自平衡
                */
                if ((sr == null || !sr.red) && (sl == null || !sl.red)) {
                    xpr.red = true;
                    x = xp;
                } else {
                    /**
                    * 该条件综合评价为：兄弟节点的右孩子为黑色
                    * 1、sr == null：兄弟的右孩子为黑色（空节点的颜色其实也是黑色）
                    * 2、!sr.red：兄弟节点的右孩子颜色为黑色
                    */
                    if (sr == null || !sr.red) {
                        /**
                        * sl != null：兄弟的左孩子是存在且颜色是红色的
                        * 情况2-2、兄弟节点右孩子为黑色、左孩子为红色
                        * 处理方式：兄弟节点的左孩子设为黑色，兄弟节点设为红色，以兄弟节点为支点进行右旋，重新设置x的兄弟节点，继续参与自平衡
                        */
                        if (sl != null)
                            sl.red = false;
                        xpr.red = true;
                        root = rotateRight(root, xpr);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    /**
                    * 情况2-3、兄弟节点的右孩子是红色
                    * 处理方式：
                    * 1、如果兄弟节点存在，兄弟节点的颜色设置为父节点的颜色
                    * 2、兄弟节点的右孩子存在，颜色设为黑色
                    * 3、如果父节点存在，将父节点的颜色设为黑色
                    * 4、以父节点为支点进行左旋
                    */
                    if (xpr != null) {
                        xpr.red = (xp == null) ? false : xp.red;
                        if ((sr = xpr.right) != null)
                            sr.red = false;
                    }
                    // 父节点不为空
                    if (xp != null) {
                        xp.red = false;
                        root = rotateLeft(root, xp);
                    }
                    // 替换节点指向根节点，平衡完成
                    x = root;
                }
            }
        } else {
            /**
            * 替换节点是父节点的右孩子节点
            * 情况1、兄弟节点存在且为红色
            * 处理方式：兄弟节点变黑，父节点变红，以父节点为支点进行左旋，重新获取兄弟节点，继续参与自平衡
            */
            if (xpl != null && xpl.red) {
                xpl.red = false;
                xp.red = true;
                root = rotateRight(root, xp);
                xpl = (xp = x.parent) == null ? null : xp.left;
            }
            // 不存在兄弟节点，x指向父节点，向上调整
            if (xpl == null)
                x = xp;
            else {
                // sl-兄弟节点的左孩子，sr-兄弟节点的右孩子
                HashMap.TreeNode<K, V> sl = xpl.left, sr = xpl.right;
                /**
                * 情况2-1：兄弟节点存在，且两个孩子的颜色均为黑色
                * 1、sr == null || !sr.red：兄弟的右孩子为黑色（空节点的颜色其实也是黑色）
                * 2、sl == null || !sl.red：兄弟的左孩子为黑色（空节点的颜色其实也是黑色）
                * 处理方式：兄弟节点为红色，替换节点指向父节点，继续参与自平衡
                */
                if ((sl == null || !sl.red) && (sr == null || !sr.red)) {
                    xpl.red = true;
                    x = xp;
                } else {
                    /**
                    * 该条件综合评价为：兄弟节点的左孩子为黑色
                    * 1、sr == null：兄弟的左孩子为黑色（空节点的颜色其实也是黑色）
                    * 2、!sr.red：兄弟节点的左孩子颜色为黑色
                    */
                    if (sl == null || !sl.red) {
                        /**
                        * sl != null：兄弟的右孩子是存在且颜色是红色的
                        * 情况2-2、兄弟节点左孩子为黑色、右孩子为红色
                        * 处理方式：兄弟节点的右孩子设为黑色，兄弟节点设为红色，以兄弟节点为支点进行左，重新设置x的兄弟节点，继续参与自平衡
                        */
                        if (sr != null)
                            sr.red = false;
                        xpl.red = true;
                        root = rotateLeft(root, xpl);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    /**
                    * 情况2-3、兄弟节点的左孩子是红色
                    * 处理方式：
                    * 1、如果兄弟节点存在，兄弟节点的颜色设置为父节点的颜色
                    * 2、兄弟节点的左孩子存在，颜色设为黑色
                    * 3、如果父节点存在，将父节点的颜色设为黑色
                    * 4、以父节点为支点进行右旋
                    */
                    if (xpl != null) {
                        xpl.red = (xp == null) ? false : xp.red;
                        if ((sl = xpl.left) != null)
                            sl.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateRight(root, xp);
                    }
                    // 替换节点指向根节点，平衡完成
                    x = root;
                }
            }
        }
    
```

