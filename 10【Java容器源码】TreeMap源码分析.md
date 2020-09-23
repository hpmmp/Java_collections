TreeMap 底层的数据结构就是红黑树，和 HashMap 的红黑树结构一样。

不同的是，TreeMap 利用了红黑树左节点小，右节点大的性质，根据 key 进行排序，使每个元素能够插入到红黑树大小适当的位置，维护了 key 的大小关系，适用于 key 需要排序的场景。

因为底层使用的是平衡红黑树的结构，所以 containsKey、get、put、remove 等方法的时间复杂度都是 log(n)。

# 1.结构
TreeMap 继承关系，核心成员变量，主要构造函数：
```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable{
    
    // 比较器，如果外部有传进来 Comparator 比较器，首先用外部的
    // 如果外部比较器为空，则使用 key 自己实现的 Comparable#compareTo 方法
    private final Comparator<? super K> comparator;

    //红黑树的根节点
    private transient Entry<K,V> root;

    //红黑树的已有元素大小
    private transient int size = 0;

    //树结构变化的版本号，用于迭代过程中的快速失败场景
    private transient int modCount = 0;
	
	//--------------------------------红黑树节点-------------------------------------
     static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent; // 相较于平时算法的树，这里多了parent
        boolean color = BLACK;

        Entry(K key, V value, Entry<K,V> parent) {}
        public K getKey() {}
        public V getValue() {}
        public V setValue(V value) {}
        public boolean equals(Object o) {}
        public int hashCode() {}
        public String toString() {}
    }
    
    //---------------------------------构造器------------------------------------
    // 传入比较器，实现
    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }
    
    // 传入map
    public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }
    
    // 空参构造
    public TreeMap() {
        comparator = null;
    }
    
    .........
}
```
TreeMap的关键是key要进行比较
  - key实现Compareable接口，重写compareTo方法
  - 自定义比较器实现Comparator
```java
public interface Comparator<T> { // T一般是Map的key
        int compare(T o1, T o2);
        boolean equals(Object obj);
    }
```
    
# 2.方法解析&api

## 2.1 增加元素：put
put 大概的步骤：

1. 如果红黑树根节点为空，新增节点直接作为根节点
2. 寻找父节点
   - 优先使用外部比较器Comparator，没有时再使用key的compareTo方法
   - 找到自己应该挂在红黑树那个节点，通过比较大小即可，红黑树左边小，右边大
   - 如果key比较相等的话，直接赋值原来的值
3. 新建节点到找到的父节点上
4. 着色旋转
```java
public V put(K key, V value) {
    	//判断红黑树的节点是否为空，为空的话，新增的节点直接作为根节点
        Entry<K,V> t = root;
        //红黑树根节点为空，直接新建
        if (t == null) {
            //key是不是null。
            //外部比较器为空的话，也判断key有没有实现Comparable
            compare(key, key); // type (and possibly null) check
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
    
    	//-----------------------------寻找父节点-------------------------------------
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) { // 如果有传入比较器
            // 找到key应该新增的位置，就是应该挂载那个节点的头上
            do {
                //当结束时，parent就是t上次比过的对象，t的值和parent的值最接近
                parent = t;
                cmp = cpr.compare(key, t.key);
                //key小于t，把t左边的值赋予t，循环再比，红黑树左边的值比较小
                if (cmp < 0)
                    t = t.left;
                //key大于t，把t右边的值赋予t，循环再比，红黑树右边的值比较大
                else if (cmp > 0)
                    t = t.right;
                //如果相等的话，直接覆盖原值
                else
                    return t.setValue(value);
                // t 为空，说明已经到叶子节点了
            } while (t != null);
        }
        else { // 没有比较器，调用key的compareTo方法
            // 若key=null ，抛出异常
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                //拿到key自己实现的比较器
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                //重复上面的步骤
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
    
    	//--------------------------------执行新增------------------------------------
        Entry<K,V> e = new Entry<>(key, value, parent);
        //cmp 代表最后一次对比的大小，小于 0 ，代表 e 在上一节点的左边
        if (cmp < 0)
            parent.left = e;
        //cmp 代表最后一次对比的大小，小于 0 ，代表 e 在上一节点的右边，相等的情况已经处理了。
        else
            parent.right = e;
    
        //--------------------------------旋转着色-------------------------------------
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
```
从源码中，我们可以看到

- 新增节点时，就是利用了红黑树左小右大的特性，从根节点不断往下查找，直到找到节点是 null 为止，节点为 null 说明到达了叶子结点；
- 查找过程中，发现 key 值已经存在，直接覆盖；
- TreeMap 是禁止 key 是 null 值的。

## 2.2 获取元素：get & getEntry

### get()
```java
public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }
```
    
### getEntry()
```java
final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        // 通过简单的查询红黑树查找到
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }
```
最后有一个问题，HashMap、TreeMap、LinkedHashMap 三者有什么相同点和不同点？

* 相同点：
	1. 三者在特定的情况下，都会使用红黑树；
	2. 底层的 hash 算法相同；
	3. 在迭代的过程中，如果 Map 的数据结构被改动，都会报 ConcurrentModificationException 的错误。
* 不同点：
	1. HashMap 数据结构以数组为主，查询非常快，TreeMap 数据结构以红黑树为主，利用了红黑树左小右大的特点，可以实现 key 的排序，LinkedHashMap 在 HashMap 的基础上增加了链表的结构，实现了插入顺序访问和最少访问删除两种策略;
	2. 由于三种 Map 底层数据结构的差别，导致了三者的使用场景的不同，TreeMap 适合需要根据 key 进行排序的场景，LinkedHashMap 适合按照插入顺序访问，或需要删除最少访问元素的场景，剩余场景我们使用 HashMap 即可，我们工作中大部分场景基本都在使用 HashMap；
	3. 由于三种 map 的底层数据结构的不同，导致上层包装的 api 略有差别。

