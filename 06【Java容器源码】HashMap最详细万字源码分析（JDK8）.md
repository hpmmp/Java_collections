从 HashMap 的类注释中，我们可以得到如下信息：

- 允许 null 值，不同于 HashTable ，是线程不安全的；
- load factor（负载因子） 默认值是 0.75， 是均衡了时间和空间损耗算出来的值，较高的值会减少空间开销（扩容减少，数组大小增长速度变慢），但增加了查找成本（hash 冲突增加，链表长度变长），不扩容的条件：数组容量 > 需要的数组大小 /load factor；
- 如果有很多数据需要储存到 HashMap 中，建议 HashMap 的容量一开始就设置成足够的大小，这样可以防止在其过程中不断的扩容，影响性能；
- HashMap 是非线程安全的，我们可以自己在外部加锁，或者通过 Collections#synchronizedMap 来实现线程安全，Collections#synchronizedMap 的实现是在每个方法上加上了 synchronized 锁；
- 在迭代过程中，如果 HashMap 的结构被修改，会快速失败。

# 1.结构
HashMap 继承关系，核心成员变量，主要构造函数：
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    	implements Map<K,V>, Cloneable, Serializable {
    	 //---------------------------------默认值---------------------------------
    	 //初始容量为 16
         static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    
         //最大容量
         static final int MAXIMUM_CAPACITY = 1 << 30;
    
         //负载因子默认值
         static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
         //（桶上的）链表长度大于等于8时，链表转化成红黑树
         static final int TREEIFY_THRESHOLD = 8;
    
         //（桶上的）红黑树大小小于等于6时，红黑树转化成链表
         static final int UNTREEIFY_THRESHOLD = 6;
    
         // 当数组容量大于 64 时，链表才会转化成红黑树
         static final int MIN_TREEIFY_CAPACITY = 64;
    	
    	//--------------------------------属性-----------------------------------
        // 负载因子，当已有槽点点占总槽点数的比值，达到后扩容
        // 可理解为，一个数组能用的槽点只用 ~%
        final float loadFactor;
        
         // 记录迭代过程中 HashMap 结构（数组）是否发生变化，如果有变化，迭代时会 fail-fast
         transient int modCount;
    
         // HashMap 的实际大小（数组大小），注意两点：
         // 1.具体节点（node）的数量并没有成员变量做记录，只是在每次遍历一个桶时用binCount作为局部变量计数
         // 2.size可能不准(因为当你拿到这个值的时候，可能又发生了变化)
         transient int size;
    
         // 存放数据的数组
         transient Node<K,V>[] table;
    
         // 阈值，两个作用：初始容量 --> 扩容阈值
         int threshold;  
         
         //-------------------------------Node---------------------------------------------
         // 链表的节点
         // 注：一个哈希桶中的node，hash和key可能都不相同，因为槽点是通过hash%(n-1)得到的
         static class Node<K,V> implements Map.Entry<K,V> {//Map.Entry是个接口
            final int hash; // 当前node的hash值，作用是决定位于哪个哈希桶，是通过key的hashCode计算来的
            final K key; // 可能是包装类实例可能是自定义对象，不同key可能有相同hash
            V value;
            Node<K,V> next; //指向当前node的下一个节点，构成单向链表
    
            Node(int hash, K key, V value, Node<K,V> next) {
                this.hash = hash;
                this.key = key;
                this.value = value;
                this.next = next;
            }
    
            public final K getKey()        { return key; }
            public final V getValue()      { return value; }
            public final String toString() { return key + "=" + value; }
    
            public final int hashCode() {
                // Objects.hashCode 防止空指针
                return Objects.hashCode(key) ^ Objects.hashCode(value);
            }
    
            public final V setValue(V newValue) {
                V oldValue = value;
                value = newValue;
                return oldValue;
            }
    
            public final boolean equals(Object o) {
                if (o == this)
                    return true;
                if (o instanceof Map.Entry) {
                    Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                    if (Objects.equals(key, e.getKey()) &&
                        Objects.equals(value, e.getValue()))
                        return true;
                }
                return false;
            }
        }
        
         // 红黑树的节点
         // 注意：这里是直接继承LinkedListMap.Entry
     	static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
     	
     	//---------------------------构造方法----------------------------------------------
     	// 传入初始化数组节点，负载因子
     	public HashMap(int initialCapacity, float loadFactor) {
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal initial capacity: " +
                                                   initialCapacity);
            // 若传入的初始化容量 > 最大，使用最大
            if (initialCapacity > MAXIMUM_CAPACITY)
                initialCapacity = MAXIMUM_CAPACITY;
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal load factor: " +
                                                   loadFactor);
            this.loadFactor = loadFactor;
            this.threshold = tableSizeFor(initialCapacity);
        }
        
        // 负载因子
        public HashMap(int initialCapacity) {
        	// 使用默认负载因子
            this(initialCapacity, DEFAULT_LOAD_FACTOR);
        }
    
        // 使用16,0.75
        public HashMap() {
            this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
        }
    }
```
这里主要关注Capacity和threshold
## 1.1 Capacity必须是2的幂

- 目的：&运算比%运算的速度更快，hash表最主要的是散列均匀，因此通过hash % n可以保证hash表是最均匀的，只有当n是2的n次幂时，hash % n 等价于 hash & (n - 1)，这样即保证了运算效率又保证了散列均匀
- 实现
  - 空参构造：16
  - 指定InitialCap：通过 threshold=tableForSize 计算出初始容量，得到 >= cap 且最接近的2的幂
    比如  给定初始化大小 19，实际上初始化大小为 32，为 2 的 5 次方
  - 后续扩容 resize() >> 1
```java
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
## 1.2 threshold阈值

threshold有两个作用：

1. 初始容量：用于指定容量时的初始化，=tableSizeFor(initialCap)，在resize()中使用
   注：1.此时thresho肯定是2的幂  2.初始化完成后就直接变成 newThr = NewCap*0.75 成为扩容阈值
2. 扩容阈值：然后用于记录数组扩容阈值，=数组容量*0.75，在putVal()中使用
   注：当threshod=Integer.MAX_VALUE无法再扩容

# 2.方法解析&api

## 2.1 hash

计算每个Node的hash值，这个hash值是确定哈希桶的依据

- 那为什么不直接拿key作为分桶依据，而是要hash值？因为不是所有的key都是Integer
- 那得到这个hash值后，怎么计算槽点？具体的槽点 = (length-1) % hash = hash % (length-1)
```java
// jdk8比jdk7引入了红黑树，所以hash方法进行了简化
static final int hash(Object key) {
        int h;
    	// null = 0
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
 }
```
从代码中看出，要注意以下两点：

1. 如果某个类型要作为hashmap的key，必须要有hashcode方法
2. 将hashCode值再右移16位是为了让算出来的hash更分散，以减少哈希冲突

## 2.2 向Hash表中添加元素
### put()
```java
 public V put(K key, V value) {
        	// 传入hash值的目的在于，定位目标桶
            return putVal(hash(key), key, value, false, true);
    }
```
### putVal()
1. 判断数组有无初始化，若无先初始化
   注：跟ArrayList一样，不是一开始就初始化好数组，而是在第一次添加元素再初始化
2. 执行新增逻辑
   - 无hash冲突：即当前位置为空，直接增加
     1. 在方法最后modCount++，记录hashmap数组发生变化
     2. if (++size > threshold)  resize(); 判断是否要扩容
        注：size记录的是数组大小，只有无hash冲突时，才会size++然后判断扩容
     3. 最后返回null
   - hash冲突下：需要找到位置e（key与hash相同节点 或 队尾），追加或判断时候覆盖旧值
     - 数组上的这个节点的与要新增节点key与hash相同
     - 红黑树，使用红黑树的方式新增
     - 链表，依次遍历，binCount记录链表长度，判断有无key与hash相同的节点
       - e != null 若onlyIfAbsent=false（覆盖旧值） ,返回oldValue
       - e == null （追加到队尾），if (binCount >= TREEIFY_THRESHOLD - 1) 判断是否需要链表转红黑树
```java
// 入参 hash：通过 hash 算法计算出来的值。用于定位目标桶
// 入参 onlyIfAbsent：absent缺席，表示只在key不存在时添加，默认为 false
// 返回的是oldValue，若不存在相同key就返回null
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // n 表示数组的长度，i 为数组索引下标，p 为 i 下标位置的 Node 值
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果数组为空，使用 resize 方法初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    //---------------------------新增逻辑--------------------------------------------
    // 计算数组下标，实际上就是 hash %（n-1）
    // 并判断如果当前索引位置是空的，直接生成新的节点在当前索引位置上
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果当前索引位置有值的处理方法，即我们常说的如何解决 hash 冲突
    else {
        // e 当前节点的临时变量
        Node<K,V> e; K k;
        // 如果 key 的 hash 和值都相等，直接把当前下标位置的 Node 值赋值给临时变量
        // 注：这里是先判断key的hash值，因为hash不同key一定不同，
        //     然后再判断key，这里还要==key 是因为key可能是缓存池的包装类
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是红黑树，使用红黑树的方式新增
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 是个链表，把新节点放到链表的尾端
        else {
            // 自旋，链表头开始遍历
            // 注：这个binCount还有记录链表长度的作用
            for (int binCount = 0; ; ++binCount) {
                // p.next == null 表明此时已经遍历到了链表最后
                if ((e = p.next) == null) {
                    // 把新节点放到链表的尾部 
                    p.next = newNode(hash, key, value, null);
                    // 当链表的长度大于等于 8 时，链表转红黑树
                    // 注：在treeifyBin方法中还要判断size是否大于64，若小于64不会树化会扩容
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 链表遍历过程中，发现有元素和新增的元素相等，结束循环
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 更改循环的当前元素，使 p 在遍历过程中，一直往后移动。
                p = e;
            }
        }
        
        // 说明链表中存在与key相同的节点
        if (e != null) {
            V oldValue = e.value;
            // 当 onlyIfAbsent 为 false 时，才会覆盖值 
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            // 返回老值
            return oldValue;
        }
    }
    
    //---------------------------------------------------------------------------------
    // 记录 HashMap 的数据结构发生了变化
    ++modCount;
    // 如果 HashMap 的实际大小大于扩容的门槛，开始扩容
    // 注：这里的逻辑是先增加再扩容，因为不一定是新增还有可能是覆盖
    if (++size > threshold)
        resize();
    
    // 对于LinkedListMap判断是否执行LRU条件之一（默认true）
    afterNodeInsertion(evict);
    return null;
}
```
## 2.3 链表树化&向红黑树中添加元素

### treeifyBin()

链表转化为红黑树
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
            int n, index; Node<K,V> e;
        	// 如果当前hash表的长度小于64，不会树化而是进行扩容
            if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
                resize();
            else if ((e = tab[index = (n - 1) & hash]) != null) {
                TreeNode<K,V> hd = null, tl = null;
                do {
                    TreeNode<K,V> p = replacementTreeNode(e, null);
                    if (tl == null)
                        hd = p;
                    else {
                        p.prev = tl;
                        tl.next = p;
                    }
                    tl = p;
                } while ((e = e.next) != null);
                if ((tab[index] = hd) != null)
                    hd.treeify(tab);
            }
        }
```
- 转换条件
  - 表长度大于等于 8，并且整个数组大小大于 64 时，才会转成红黑树
  - 当数组大小小于 64 时，只会触发扩容，不会转化成红黑树
- 链表长度阈值为什么是8？
  链表查询的时间复杂度是O (n)，红黑树的查询复杂度是 O (log (n))。在链表数据不多的时候，使用链表进行遍历也比较快，只有当链表数据比较多的时候，才会转化成红黑树，但红黑树需要的占用空间是链表的 2 倍，考虑到转化时间和空间损耗，所以我们需要定义出转化的边界值。
  在考虑设计 8 这个值的时候，参考了泊松分布概率函数，由泊松分布中得出结论，链表各个长度的命中概率为：
  
      * 0:    0.60653066
      * 1:    0.30326533
      * 2:    0.07581633
      * 3:    0.01263606
      * 4:    0.00157952
      * 5:    0.00015795
      * 6:    0.00001316
      * 7:    0.00000094
      * 8:    0.00000006
  意思是，当链表的长度是 8 的时候，出现的概率是 0.00000006，不到千万分之一，所以说正常情况下，链表的长度不可能到达 8 ，而一旦到达 8 时，肯定是 hash 算法出了问题，所以在这种情况下，为了让 HashMap 仍然有较高的查询性能，所以让链表转化成红黑树，我们正常写代码，使用 HashMap 时，几乎不会碰到链表转化成红黑树的情况，毕竟概念只有千万分之一。

### putTreeVal()

1. 首先判断新增的节点在红黑树上是不是已经存在，判断手段有如下两种：
   1. 如果节点没有实现 Comparable 接口，使用 equals 进行判断；
   2. 如果节点自己实现了 Comparable 接口，使用 compareTo 进行判断。
2. 新增的节点如果已经在红黑树上，直接返回；不在的话，判断新增节点是在当前节点的左边还是右边，左边值小，右边值大；
3. 自旋递归 1 和 2 步，直到当前节点的左边或者右边的节点为空时，停止自旋，当前节点即为我们新增节点的父节点；
4. 把新增节点放到当前节点的左边或右边为空的地方，并于当前节点建立父子节点关系；
5. 进行着色和旋转，结束。
```java
//入参 h：key 的hash值
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    //找到根节点
    TreeNode<K,V> root = (parent != null) ? root() : this;
    //自旋
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // p hash 值大于 h，说明 p 在 h 的右边
        if ((ph = p.hash) > h)
            dir = -1;
        // p hash 值小于 h，说明 p 在 h 的左边
        else if (ph < h)
            dir = 1;
        //要放进去key在当前树中已经存在了(equals来判断)
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        //自己实现的Comparable的话，不能用hashcode比较了，需要用compareTo
        else if ((kc == null &&
                  //得到key的Class类型，如果key没有实现Comparable就是null
                  (kc = comparableClassFor(k)) == null) ||
                  //当前节点pk和入参k不等
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        //找到和当前hashcode值相近的节点(当前节点的左右子节点其中一个为空即可)
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            //生成新的节点
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            //把新节点放在当前子节点为空的位置上
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            //当前节点和新节点建立父子，前后关系
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            //balanceInsertion 对红黑树进行着色或旋转，以达到更多的查找效率，着色或旋转的几种场景如下
            //着色：新节点总是为红色；如果新节点的父亲是黑色，则不需要重新着色；		   	
            //如果父亲是红色，那么必须通过重新着色或者旋转的方法，再次达到红黑树的5个约束条件
            //旋转： 父亲是红色，叔叔是黑色时，进行旋转
            //如果当前节点是父亲的右节点，则进行左旋
            //如果当前节点是父亲的左节点，则进行右旋
          
            //moveRootToFront 方法是把算出来的root放到根节点上
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}	
```

红黑树的 5 个原则：

- 节点是红色或黑色
- 根是黑色
- 所有叶子都是黑色
- 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点
- 从每个叶子到根的所有路径上不能有两个连续的红色节点

## 2.3 扩容resize

resize()有两个作用，1.初始化数组，2.扩容。

1. 计算新数组大小：
   - 初始化
     - 指定容量，newCap=threshold
     - 未指定容量，newCap=16
   - 扩容：newCap=oldCap * 2
2. 执行扩容策略：逐个遍历所有槽点
   - 当前槽点只有一个node，直接重新分配到新数组即可newTab[e.hash & (newCap - 1)]
   - 当前槽点下是红黑树
   - 当前槽点下是链表：因为链表中所有node的hash和key相同，而现在数组扩容了两倍，所以现在的想法是将当前链等分成两部分
     1. 怎么等分成两部分？分成high链与low链，两链关系可近似理解为单双数节点
     2. 怎么实现？Head用来标识链首，Tail用来尾连接
     3. 两链放在新数组哪里？：low链置于newTab[ j ]，high链置于newTab[ j + oldCap ]（ j 表示在原来数组位置）
```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
    	// 情况一：oldCap>0表示要扩容
        if (oldCap > 0) {
            // 当老数组容量已达最大值时无法再扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // Cap * 2 ， Thr * 2
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
    	// 情况二：初始化，且已指定初始化容量
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
    	// 情况三：初始化，未指定初始化容量
        else {               
            // 以默认初始化容量16进行初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            // 16 * 0.75 = 12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	// 用指定容量初始化后，更新其扩容阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
    	// 执行初始化，建立新数组
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    	// 将新数组附给table
        table = newTab;
    	
    	//-------------------------------扩容策略-----------------------------------------------
    	// 若是执行的初始化，old=null，就不用走这里的扩容策略了
        if (oldTab != null) {
            // 遍历原数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 当原数组的当前槽点有值时,将其赋给e
                if ((e = oldTab[j]) != null) {
                    // 释放原数组当前节点内存，帮助GC
                    oldTab[j] = null;
                    // 若当前槽点只有一个值，直接找到新数组相应位置并赋值
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 若当前节点下是红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 当前节点下是链表
                    else { // preserve order
                        // 将该链表分成两条链，low低位链 和 high高位链
                        // Head标识链头，Tail用来连接
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // do while((e = next) != null)
                        do {
                            next = e.next;
                            // 低位链连接
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 高位链连接
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 将低位链放置于老链同一下标
                        // eg.原来当前节点位于oldTab[2],则现在低位链还置于newTab[2]
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 将高位链放置于老链下标+oldCap
                        // eg.原来当前节点位于oldTab[2],则现在高位链置于newTab[2+8=10]
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

## 2.4 从Hash表中获取元素

### get()

计算hash，并调用getNode
```java
public V get(Object key) {
            Node<K,V> e;
        	// 如果是null直接返回null
            return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

```
### getNode()

1. 根据hash值定位数组的索引位置
2. equals 判断当前节点是否是我们需要寻找的 key
   - 是的话直接返回
   - 不是则判断当前节点有无 next 节点，有的话判断是链表类型，还是红黑树类型。
3. 分别走链表和红黑树不同类型的查找方法
```java
// 传入的hash值用于确定哈系桶，key用于确定具体节点
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 数组不为空 && hash算出来的索引下标有值，
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // hash 和 key 的 hash 相等，直接返回
            if (first.hash == hash &&
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // hash不等，看看当前节点的 next 是否有值
            if ((e = first.next) != null) {
                // 使用红黑树的查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 采用自旋方式从链表中查找 key，e 为链表的头节点
                do {
                    // 如果当前节点 hash == key 的 hash，并且 equals 相等，当前节点就是我们要找的节点
                    // 先比较hash效率高，因为hash一定是数字，key可能是包装类可能是自定义对象
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                    // 否则，把当前节点的下一个节点拿出来继续寻找
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
### find()

红黑树寻找指定节点

1. 从根节点递归查找；
2. 根据 hashcode，比较查找节点，左边节点，右边节点之间的大小，根本红黑树左小右大的特性进行判断；
3. 判断查找节点在第 2 步有无定位节点位置，有的话返回，没有的话重复 2，3 两步；
4. 一直自旋到定位到节点位置为止。
```java
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
            TreeNode<K,V> p = this;
            do {
                int ph, dir; K pk;
                TreeNode<K,V> pl = p.left, pr = p.right, q;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.find(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
            return null;
        }
```

## 2.5 迭代器

Map 对 key、value 和 entity（节点） 都提供了迭代器。这些迭代器都是通过map.~Set().iterator()进行调用

- 迭代key：HashMap --keySet()--> KeySet --iterator()--> KeyIterator
- 迭代value：HashMap --values()--> Values--iterator()--> ValueIterator
- 迭代key：HashMap --entrySet()--> EntrySet--iterator()--> EntryIterator

虽然是不同的迭代器，但是它们本质上却没有区别：

1. 都继承了HashIterator
2. 都只有一个方法：next()，而且里面调用的都是 HashIterator.nextNode()，只不过最后在node中取值不同
```java
final class KeyIterator extends HashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().key; } // 调用父类的nextNode方法，返回node的key
}

final class ValueIterator extends HashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; } // 调用父类的nextNode方法，返回node的value
}

final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }  // 调用父类的nextNode方法，返回node
}
```
### HashIterator
```java
 abstract class HashIterator {
            Node<K,V> next;        // next entry to return
            Node<K,V> current;     // current entry
            int expectedModCount;  // for fast-fail
            int index;             // current slot
    
            HashIterator() {
                expectedModCount = modCount;
                Node<K,V>[] t = table;
                current = next = null;
                index = 0;
                if (t != null && size > 0) { // advance to first entry
                    do {} while (index < t.length && (next = t[index++]) == null);
                }
            }
    ......
        }
```
### hasNext()
判断当前node在桶中是否有下一个node
```java
 public final boolean hasNext() {
        return next != null;
    }
```
### nextNode()

获取当前节点的下一个node。

- 整体的迭代策略是逐个桶遍历，可理解成外层是遍历数组，内层是遍历链表（红黑树）
- 该方法屏蔽了node处于不同桶所带来的差异，就好像所有元素在一个桶中。
```java
final Node<K,V> nextNode() {
    Node<K,V>[] t;
    // 记录next结点
    Node<K,V> e = next; 
    // 若在遍历时对HashMap进行结构性的修改则会抛出异常
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    // 下一个结点为空，抛出异常
    if (e == null)
        throw new NoSuchElementException();
    // 如果下一个结点为空 && table不为空，表示当前桶中所有结点已经遍历完
    // 注：核心！！！实现了跨桶遍历
    if ((next = (current = e).next) == null && (t = table) != null) {
        // 寻找下一个不为空的桶：未到最后一个槽点 && 下一个槽点不为空
        do {} while (index < t.length && (next = t[index++]) == null);
    }
    return e;
}
```
### remove()
```java
public final void remove() {
        Node<K,V> p = current;
        // 当前结点为空，抛出异常
        if (p == null)
            throw new IllegalStateException();
        // 若在遍历时对HashMap进行结构性的修改则会抛出异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 若在遍历时对HashMap进行结构性的修改则会抛出异常
        current = null;
        K key = p.key;
        // 移除结点
        removeNode(hash(key), key, null, false, false);
        // 赋最新值
        expectedModCount = modCount;
    }
```
   
