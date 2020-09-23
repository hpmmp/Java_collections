看源码先看类注释上，我们可以得到的信息有：

1. 底层实现基于 HashMap，所以迭代时不能保证按照插入顺序，或者其它顺序进行迭代；
2. add、remove、contanins、size 等方法的耗时性能，是不会随着数据量的增加而增加的，这个主要跟 HashMap 底层的数组数据结构有关，不管数据量多大，不考虑 hash 冲突的情况下，时间复杂度都是 O (1)；
3. 线程不安全的，如果需要安全请自行加锁，或者使用 Collections.synchronizedSet；
4. 迭代过程中，如果数据结构被改变，会快速失败的，会抛出 ConcurrentModificationException 异常。

# 1.结构
HashSet 继承关系，核心成员变量，主要构造函数：
```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable{
    
    // 把 HashMap 组合进来，key 是 Hashset 的 key，value 是下面的 PRESENT
    private transient HashMap<E,Object> map;

    // HashMap 中的 value，所有node中的value相同
    private static final Object PRESENT = new Object();
    
    //---------------------------构造方法---------------------------------------
    // 直接初始化一个HashMap
    public HashSet() {
        map = new HashMap<>();
    }
    
    // 对 HashMap 的容量进行了计算，在 16 和 给定值大小之间选择最大的值
    public HashSet(Collection<? extends E> c) {
    	// 选取最优初始容量
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
}
```

## 1.1 Set的实现就是基于Map的key的唯一性

- 因为Map的key如果相同，必须选择是否覆盖，故不存在相等的key
- 而Map的value，在Set中大家都是相同的（即PRESENT）

## 1.2 HashSet 是如何组合 HashMap

刚才是从类注释中看到，HashSet 的实现是基于 HashMap 的，在 Java 中，要基于基础类进行创新实现，有两种办法：

- 继承基础类，覆写基础类的方法，比如说继承 HashMap , 覆写其 add 的方法；
- 组合基础类，通过调用基础类的方法，来复用基础类的能力。

**HashSet 使用的就是组合 HashMap**，其优点如下：

- 继承表示父子类是同一个事物，而 Set 和 Map 本来就是想表达两种事物，所以继承不妥，而且 Java 语法限制，子类只能继承一个父类，后续难以扩展。
- 组合更加灵活，可以任意的组合现有的基础类，并且可以在基础类方法的基础上进行扩展、编排等，而且方法命名可以任意命名，无需和基础类的方法名称保持一致。

如果碰到类似问题，我们的原则也是尽量**多用组合，少用继承**。

## 1.3 最优容量初始化

上述代码中：Math.max ((int) (c.size ()/.75f) + 1, 16)，就是对 HashMap 的容量进行了计算，翻译成中文就是 取括号中两个数的最大值（期望的值 / 0.75+1，默认值 16），从计算中，我们可以看出 HashSet 的实现者对 HashMap 的底层实现是非常清楚的，主要体现在两个方面：

- 和 16 比较大小的意思是说，如果给定 HashMap 初始容量小于 16 ，就按照 HashMap 默认的 16 初始化好了，如果大于 16，就按照给定值初始化。
- HashMap 扩容的伐值的计算公式是：Map 的容量 * 0.75f，一旦达到阀值就会扩容，此处用 (int) (c.size ()/.75f) + 1 来表示初始化的值，这样使我们期望的大小值正好比扩容的阀值还大 1，就不会扩容，符合 HashMap 扩容的公式。

可想到**HashMap 初始化大小值的模版公式：取括号内两者的最大值（期望的值 / 0.75+1，默认值 16)**，因为虽然要用期望值个槽点，但在数组中能用的槽点只占总数0.75，所以要 / 0.75，+1是避免刚好达到0.75的情况；

# 2.方法解析&api

HashSet 的其他方法就比较简单了，就是对 Map 的 api 进行了一些包装

## 2.1 add

add就是对HashMap的put做简单包装
```java
public boolean add(E e) {
    // 直接使用 HashMap 的 put 方法，进行一些简单的逻辑判断
    // 注：这里所有value都是PRESENT
    return map.put(e, PRESENT)==null;
}
```
## 2.2 remove
```java
public boolean remove(Object o) {
    	// 只有当o存在时，才能删除成功
    	// 而在HashMap中，o是key，当key存在时再删除才会返回value
        return map.remove(o)==PRESENT;
}
```

## 2.3 iterator

迭代器，直接返回HashMap的key迭代器，因为HashMap的key组成的Set

  ```java
  public Iterator<E> iterator() {
        return map.keySet().iterator();
}
  ```

最后，HashSet 具体实现值得我们借鉴的地方

- 对组合还是继承的分析和把握；
- 对复杂逻辑进行一些包装，使吐出去的接口尽量简单好用；
- 组合其他 api 时，尽量多对组合的 api 多些了解，这样才能更好的使用 api；
- HashMap 初始化大小值的模版公式：取括号内两者的最大值（期望的值 / 0.75+1，默认值 16）
