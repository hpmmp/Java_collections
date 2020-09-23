## 1.HashMap 底层数据结构是什么？
答：HashMap 底层是数组 + 链表 + 红黑树的数据结构

- 数组的主要作用是方便快速查找，时间复杂度是 O(1)，默认大小是 16，数组的下标索引是通过 key 的 hashcode 计算出来的，数组元素叫做 Node
- 当多个 key 的 hashcode 一致，但 key 值不同时，单个 Node 就会转化成链表，链表的查询复杂度是 O(n)
- 当链表的长度大于等于 8 并且数组的大小超过 64 时，链表就会转化成红黑树，红黑树的查询复杂度是 O(log(n))，简单来说，最坏的查询次数相当于红黑树的最大深度。

## 2.为解决 hash 冲突，大概有哪些办法？
答：
1. 好的 hash 算法
2. 自动扩容，当数组大小快满的时候，采取自动扩容，可以减少 hash 冲突;
3. hash 冲突发生时，采用链表来解决;
4. hash 冲突严重时，链表会自动转化成红黑树，提高遍历速度。

## 3.HashMapHashMap 是如何扩容的？
- 扩容时机：
  1. 初始化：put 时，发现数组为空，进行初始化扩容，默认扩容大小为 16;
  2. 扩容：put 成功后，发现现有数组大小大于扩容的门阀值时，进行扩容，扩容为老数组大小的 2 倍;
- 扩容的阀值是 threshold，每次扩容时 threshold 都会被重新计算，门阀值等于数组的大小 * 影响因子（0.75）。
- 新数组初始化之后，需要将老数组的值拷贝到新数组上，链表和红黑树都有自己拷贝的方法。

## 4. hash 冲突时怎么办？

答：hash 冲突指的常出现于不同的 key 计算得到相同的 hashcode 情况。

- 如果桶中元素原本只有一个或已经是链表了，新增元素直接追加到链表尾部；
- 如果桶中元素已经是链表，并且链表个数大于等于 8 时，此时有两种情况：
  1. 如果此时数组大小小于 64，数组再次扩容，链表不会转化成红黑树;
  2. 如果数组大小大于 64 时，链表就会转化成红黑树。

这里不仅仅判断链表个数大于等于 8，还判断了数组大小，数组容量小于 64 没有立即转化的原因，猜测主要是因为红黑树占用的空间比链表大很多，转化也比较耗时，所以数组容量小的情况下冲突严重，我们可以先尝试扩容，看看能否通过扩容来解决冲突的问题。

## 5.为什么链表个数大于等于 8 时，链表要转化成红黑树了？

答：这实际是两个问题
1. 为什么要转换成红黑树？
	当链表个数太多了，遍历可能比较耗时，转化成红黑树，可以使遍历的时间复杂度降低。但转化成红黑树会有空间和转化耗时的成本。
2. 为什么是节点个数大于等于8？
	通过泊松分布公式计算，正常情况下，链表个数出现 8 的概念不到千万分之一，所以说正常情况下，链表都不会转化成红黑树，这样设计的目的，是为了防止非正常情况下，比如 hash 算法出了问题时，导致链表个数轻易大于等于 8 时，仍然能够快速遍历。

## 6.红黑树什么时候转变成链表?
答：当节点的个数小于等于 6 时，红黑树会自动转化成链表，主要还是考虑红黑树的空间成本问题，当节点个数小于等于 6 时，遍历链表也很快，所以红黑树会重新变成链表。

## 7.HashMap 在 put 时，如果数组中已经有了这个 key，我不想把 value 覆盖怎么办？取值时，如果得到的 value 是空时，想返回默认值怎么办？

答：
* 如果数组有了 key，但不想覆盖 value ，可以选择 putIfAbsent 方法，这个方法有个内置变量 onlyIfAbsent，内置是 true ，就不会覆盖，我们平时使用的 put 方法，内置 onlyIfAbsent 为 false，是允许覆盖的。

* 取值时，如果为空，想返回默认值，可以使用 getOrDefault 方法，方法第一参数为 key，第二个参数为你想返回的默认值，如 map.getOrDefault(“2”,“0”)，当 map 中没有 key 为 2 的值时，会默认返回 0，而不是空。

## 8.通过以下代码进行删除，是否可行？
```java
HashMap<String,String > map = Maps.newHashMap();
map.put("1","1");
map.put("2","2");
map.forEach((s, s2) -> map.remove("1"));
```
答：不行，会报错误 ConcurrentModificationException，forEach源码如下：
```java
public void forEach(BiConsumer<? super K, ? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount; // 开始循环之前modCount被赋值给mc
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key, e.value);
            }
            if (modCount != mc) // 删除时remove方法会修改modCount，但mc没变
                throw new ConcurrentModificationException();
        }
    }
```
建议使用迭代器的方式进行删除，原理同 ArrayList 迭代器原理，

## 9.描述一下 HashMap get、put 的过程
答：具体请参考[【Java容器源码】HashMap最详细万字源码分析（JDK8）](https://blog.csdn.net/weixin_43935927/article/details/108501752)，想不清楚的话，可以画一画。
