# 1.List区别
## 1.1 ArrayList（改动小）

ArrayList 无参初始化时，Java 7 是直接初始化 10 的大小，Java 8 去掉了这个逻辑，初始化时是空数组，在第一次 add 时才开始按照 10 进行扩容，下图是源码的差异对比图：
<img src="https://img1.sycdn.imooc.com/5d7079470001ebfd20080254.png">
List 其它方面 java7 和 8 并没有改动。

# 2.Map区别

## 2.1 HashMap（改动大）

1. 和 ArrayList 一样，Java 8 中 HashMap 在无参构造器中，丢弃了 Java 7 中直接把数组初始化 16 的做法，而是采用在第一次新增的时候，才开始扩容数组大小；
2. hash 算法计算公式不同，Java 8 的 hash 算法更加简单，代码更加简洁；
3. Java 8 的 HashMap 增加了红黑树的数据结构，这个是 Java 7 中没有的，Java 7 只有数组 + 链表的结构，Java 8 中提出了数组 + 链表 + 红黑树的结构，一般 key 是 Java 的 API 时，比如说 String 这些 hashcode 实现很好的 API，很少出现链表转化成红黑树的情况，因为 String 这些 API 的 hash 算法够好了，只有当 key 是我们自定义的类，而且我们覆写的 hashcode 算法非常糟糕时，才会真正使用到红黑树，提高我们的检索速度。
4. 也是因为 Java 8 新增了红黑树，所以几乎所有操作数组的方法的实现，都发生了变动，比如说 put、remove 等操作，可以说 Java 8 的 HashMap 几乎重写了一遍，所以 Java 7 的很多问题都被 Java 8 解决了，比如扩容时极小概率死锁，丢失数据等等。
5. 新增了一些好用的方法，
   - 比如 getOrDefault，我们看下源码，非常简单：
	```java
	// 如果 key 对应的值不存在，返回期望的默认值 defaultValue
	public V getOrDefault(Object key, V defaultValue) {
	    Node<K,V> e;
	    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
	}
	```
	2. 还有 putIfAbsent(K key, V value) 方法，意思是，如果 map 中存在 key 了，那么 value 就不会覆盖，如果不存在 key 新增成功。
	3. 还有 compute 方法，意思是允许我们把 key 和 value 的值进行计算后，再 put 到 map 中，为防止 key 值不存在造成未知错误，map 还提供了 computeIfPresent 方法，表示只有在 key 存在的时候，才执行计算，demo 如下：
```java
@Test
  public void compute(){
    HashMap<Integer,Integer> map = Maps.newHashMap();
    map.put(10,10);
    log.info("compute 之前值为：{}",map.get(10));
    map.compute(10,(key,value) -> key * value);
    log.info("compute 之后值为：{}",map.get(10));
    // 还原测试值
    map.put(10,10);

    // 如果为 11 的 key 不存在的话，需要注意 value 为空的情况，下面这行代码就会报空指针
    //  map.compute(11,(key,value) -> key * value);
    
    // 为了防止 key 不存在时导致的未知异常，我们一般有两种办法
    // 1：自己判断空指针
    map.compute(11,(key,value) -> null == value ? null : key * value);
    // 2：computeIfPresent 方法里面判断
    map.computeIfPresent(11,(key,value) -> key * value);
    log.info("computeIfPresent 之后值为：{}",map.get(11));
  }
```
结果是：
```
compute 之前值为：10
compute 之后值为：100
computeIfPresent 之后值为：null（这个结果中，可以看出，使用 computeIfPresent 避免了空指针）
```
## 2.2 LinkedHashMap
由于 Java 8 的底层数据有变动，导致 HashMap 操作数据的方法几乎重写，也使 LinkedHashMap 的实现名称上有所差异，原理上都相同，我们看下面的图，左边是 Java 7，右边是 Java 8。
<img src="https://img1.sycdn.imooc.com/5d7078fe0001a6b724721322.png">
从图中，我们发现 LinkedHashMap 的方法名有所修改，底层的实现逻辑其实都差不多的。

# 3.其他区别
Arrays 提供了很多 parallel 开头的方法。

Java 8 的 Arrays 提供了一些 parallel 开头的方法，这些方法支持并行的计算，在数据量大的时候，会充分利用 CPU ，提高计算效率，比如说 parallelSort 方法，方法底层有判断，只有数据量大于 8192 时，才会真正走并行的实现，在实际的实验中，并行计算的确能够快速的提高计算速度。

# 4.几个问题

1. Java 8 在 List、Map 接口上新增了很多方法，为什么 Java 7 中这些接口的实现者不需要强制实现这些方法呢？
	答：主要是因为这些新增的方法被 default 关键字修饰了，default 一旦修饰接口上的方法，我们需要在接口的方法中写默认实现，并且子类无需强制实现这些方法，所以 Java 7 接口的实现者无需感知。

2. Java 8 中有新增很多实用的方法，都有什么呢？
	答：比如说 getOrDefault、putIfAbsent、computeIfPresent 方法等等，具体使用细节参考上文。
3. 说说 computeIfPresent 方法的使用姿势？
	答：computeIfPresent 是可以对 key 和 value 进行计算后，把计算的结果重新赋值给 key，并且如果 key 不存在时，不会报空指针，会返回 null 值。
4. Java 8 集合新增了 forEach 方法，和普通的 for 循环有什么不同？
	答：新增的 forEach 方法的入参是函数式的接口，比如说 Consumer 和 BiConsumer，这样子做的好处就是封装了 for 循环的代码，让使用者只需关注实现每次循环的业务逻辑，简化了重复的 for 循环代码，使代码更加简洁，普通的 for 循环，每次都需要写重复的 for 循环代码，forEach 把这种重复的计算逻辑吃掉了，使用起来更加方便。
5. HashMap 8 和 7 有什么区别？
答：HashMap 8 和 7 的差别太大了，新增了红黑树，修改了底层数据逻辑，修改了 hash 算法，几乎所有底层数组变动的方法都重写了一遍，可以说 Java 8 的 HashMap 几乎重新了一遍。
