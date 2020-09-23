HashMap 是无序的，TreeMap 可以按照 key 进行排序，那有木有 Map 是可以维护插入的顺序的呢？接下来我们一起来看下 LinkedHashMap。

LinkedHashMap 本身是继承 HashMap 的，所以它拥有 HashMap 的所有特性，再此基础上，还提供了两大特性：

- 按照插入顺序进行访问；
- 实现了访问最少最先删除功能，其目的是把很久都没有访问的 key 自动删除。

# 1.结构
LinkedHashMap 继承关系，核心成员变量，主要构造函数：
```java
// LinkedHashMap继承了HashMap
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>{

	// 链表头
    transient LinkedHashMap.Entry<K,V> head;

    // 链表尾
    // 如果head=tail=null，则链表为空
    transient LinkedHashMap.Entry<K,V> tail;
	
    //--------------------------------Node-----------------------------------------
    // 在LinkedHashMap的节点叫Entry，继承HashMap的Node
    static class Entry<K,V> extends HashMap.Node<K,V> {
        // 为数组的每个元素增加了 before 和 after 属性
        Entry<K,V> before, after;
        // 但是构造函数没变，没有将链表的before和after加进去
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    // 控制LRU是否开启的元素之一，表示是否允许在访问（get）后，将当前节点移动到链表尾部，默认是false
    // 注：另一个开启元素是put时的删除策略是否允许删除头节点，默认返回false
    final boolean accessOrder;
    
    //-------------------------------构造函数-------------------------------------
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false; // 将accessOrder置为false
    }	
    
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }
    
    public LinkedHashMap() {
        super();
        accessOrder = false;
    } 
    
    // 可以通过此构造方法设置 accessOrder，即要开启LRU策略必须通过该构造函数
     public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
}
```
从上述 Map 新增的属性可以看到，LinkedHashMap 的数据结构很像是把 LinkedList 的每个元素换成了 HashMap 的 Node，像是两者的结合体，也正是因为增加了这些结构，从而能把 Map 的元素都串联起来，形成一个链表，而链表就可以保证顺序了，就可以维护元素插入进来的顺序。

LinkedHashMap 是通过双向链表和散列表这两种数据结构组合实现的，其中的“Linked”实际上是指的是双向链表，并非指用链表法解决散列冲突。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200910140313621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)

# 2.方法解析&api
## 2.1 增加元素时实现LRU

LinkedHashMap 本身是没有 put 方法实现的，由于继承了HashMap，所以会调用HashMap的 put 与 putVal 方法。但仅仅这样是不够的还要考虑LinkedHashMap的2个特性

1. LinkedHashMap要有链表特性，所以还要将新加入的节点尾插到链表中
2. LinkedHashMap可以实现LRU，原因是已经实现链表，但需要考虑以下两个问题：
   1. 具体实现策略是什么？要从 get 和 put 两方面入手
      - get：当每访问一个节点（get）后，就将当前节点置于链表尾部
        - 重置于尾部的原因：新节点尾插到链表，意味着尾部本来就新
        - 这也等同于越往前的节点越久，且没访问
      - put：当插入一个节点（put）后，且满足所有删除条件，就删除头节点
        - 这里的一定条件其实就是删除策略，一般需要用户自定义，比如size>3等
   2. 如何设置开启执行LRU？同样要从get和put入手，但默认是关闭的
      - get：要将 accessOrder 置为 true，表示同意get过的节点移到链表最后，默认是false，需构造时开启
      - put：removeEldestEntry方法要能返回true，表示删除策略允许删除，默认是false，一般需要重写
        注：其实还有两个因素 evict=true（put时默认），队列不为空（一般不涉及）

### put()
```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}
```
### putVal()
```java
// evict:决定是否删除最少访问元素，默认true
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
       .......
        // 调用newNode新增节点，但在LInkedHashMap中，重写了newNode方法，因为节点
        tab[i] = newNode(hash, key, value, null);
       ......
        // 在HashMap中是空实现，但linkedListMap中具体实现了
       afterNodeInsertion(evict);
 }
```
这里本质是依赖倒置的设计原则，上层定义接口，下层实现

### newNode()

重写HashMap的newNode方法，新增节点Entry，并调用 linkNodeLast 追加到链表的尾部
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // 新增节点，这里是new LinkedHash.Entry
    LinkedHashMap.Entry<K,V> p = new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    // 追加到链表的尾部
    linkNodeLast(p);
    return p;
}
```
### linkNodeLast()

因为LinkedHashMap还有链表特性，所以不能只是HashMap那样添加到哈希表上，而是还要添加到链表上
```java
// 入参是新创建的节点
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail; // 暂存tail    
    tail = p; // 将tail置为新节点p
    
    if (last == null) // last 为空，说明链表为空，所以还要将head设为新节点p
        head = p;    
    else { // 链表非空，直接建立新增节点和上个尾节点之间的前后关系即可
        p.before = last;
        last.after = p;
    }
}
```
### afterNodeInsertion()

afterNodeInsertion方法在HashMap中是空实现，在LinkedHashMap中是为了实现LRU策略。
```java
void afterNodeInsertion(boolean evict) { 
    LinkedHashMap.Entry<K,V> first;
    // 要删除头节点要满足三个条件
    // 1.evict=true，在put方法中默认是true
    // 2.队列不为空，但一般不牵扯这个问题，因为节点加入后才会执行该方法
    // 3.删除策略策略允许删除，默认false
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true); // removeNode 删除头节点
    }
}
```
### removeEldestEntry()

removeEldestEntry一般用于自定义删除策略，返回true表示执行删除头结点，返回false表示不删除

- 该方法一般需要用户重写，比如 return size>3时删除
- 在LinkedHashMap中提供了默认实现，即直接返回false（不删除，也可以理解成默认无法执行lru）
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
}
```
## 2.2 获取元素时实现LRU
### get()

LinkedHashMap重写了HashMap的get方法：

1. 获取具体节点还是调用HashMap的getNode方法
2. 只不过在获取到node后，加入了移动node到队尾以实现LRU的操作
```java
public V get(Object key) {
        Node<K,V> e;
        // 调用 HashMap.getNode()获取具体节点
        if ((e = getNode(hash(key), key)) == null)
            return null;
        // 如果设置了 LRU 策略
        if (accessOrder)
        // 这个方式把当前 key 移动到队尾
            afterNodeAccess(e);
        return e.value;
    }
```
不仅仅是 get 方法，执行 getOrDefault、compute、computeIfAbsent、computeIfPresent、merge 方法时，也会不断的把经常访问的节点移动到队尾，那么靠近队头的节点，自然就是很少被访问的元素了。

### afterNodeAccess()

将访问的节点移到队尾
```java
void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        // accessOrder 为 true，并且 e 不为队尾的时候
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
### LRU示例
```java
public void testAccessOrder() {
  // 构建 LinkedHashMap，这里必须使用唯一的能设置accessOrder的构造函数
  LinkedHashMap<Integer, Integer> map = new LinkedHashMap<Integer, Integer>(4,0.75f,true) {
  // 注：可以在构造函数后初始化（必须加大括号）或重写方法（直接重写）， new A() { {初始化} / 重写方法 };
    { 
      put(10, 10); // 注：调用容器具体函数初始化时，必须再加{}
      put(9, 9);
      put(20, 20);
      put(1, 1);
    }

    @Override
    // 覆写了删除策略的方法，允许返回true；即当已经有大于3个槽点被使用时，开始删除链表头结点
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
      return size() > 3;
    }
  };

  log.info("初始化：{}",JSON.toJSONString(map));
  Assert.assertNotNull(map.get(9));
  log.info("map.get(9)：{}",JSON.toJSONString(map));
  Assert.assertNotNull(map.get(20));
  log.info("map.get(20)：{}",JSON.toJSONString(map));
}
```
打印出的结果如下
```
初始化：{9:9,20:20,1:1}
map.get(9)：{20:20,1:1,9:9}
map.get(20)：{1:1,9:9,20:20}
```
可以看到，map 初始化的时候，我们放进去四个元素，但结果只有三个元素，10 不见了，这个主要是因为我们覆写了 removeEldestEntry 方法，我们实现了如果 map 中元素个数大于 3 时，我们就把队头的元素删除，当 put(1, 1) 执行的时候，正好把队头的 10 删除，这个体现了达到我们设定的删除策略时，会自动的删除头节点。

当我们调用 map.get(9) 方法时，元素 9 移动到队尾，调用 map.get(20) 方法时， 元素 20 被移动到队尾，这个体现了经常被访问的节点会被移动到队尾。

## 2.3 迭代器：基于链表

Map 对 key、value 和 entity（节点） 都提供了迭代器。这些迭代器都是通过map.~Set().iterator()进行调用

- 迭代key：LinkedHashMap --keySet()--> LinkedKeySet --iterator()--> LinkedKeyIterator
- 迭代value：LinkedHashMap --values()--> LInkedValues--iterator()--> LinkedValueIterator
- 迭代key：LinkedHashMap --entrySet()--> LinkedEntrySet--iterator()--> LinkedEntryIterator

### Linked~~Iterator

虽然是不同的迭代器，但是它们本质上却没有区别：

1. 都继承了LinkedHashIterator
2. 都只有一个方法：next()，而且里面调用的都是 LinkedHashIterator.nextNode()，只不过最后在node中取值不同
```java
final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().getKey(); } // 调用父类nextNode方法，返回node中的Key
}

final class LinkedValueIterator extends LinkedHashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; } // 调用父类nextNode方法，返回node中的value
}

final class LinkedEntryIterator extends LinkedHashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); } // 调用父类nextNode方法，直接返回node
}
```
### LinkedHashIterator

LinkedHashIterator和HashIterator的比较

1. 不同点：nextNode方法，也即迭代策略
   - HashIterator的迭代策略是：逐桶遍历，可理解成外层是遍历数组，内层是遍历链表（红黑树）
   - LinkedHashIterator：只用按照链表从头到尾遍历所有node即可
2. 相同点：hasNext方法和remove方法完全相同，所以下面代码也就没有列举这两个方法

但是对于LinkedHashMap而言，他继承了HashMap，所以这两种迭代器都能使用
```java
abstract class LinkedHashIterator {
     
        LinkedHashMap.Entry<K,V> next; // 下一个节点
        LinkedHashMap.Entry<K,V> current; // 当前节点
        int expectedModCount;  // 目标操作数（即开始迭代前的操作数）

        // 初始化时，默认从头节点开始访问
        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }
     	
     	// 核心方法nextNode()，根据链表进行迭代！！
     	final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;
            // 校验 如果迭代过程中对Map进行操作，报错
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            // 通过链表的 after 结构，找到下一个迭代的节点
            next = e.after; 
            return e;
		}
}
```
