下面列出了所有集合的类图：
![图片描述](https://img-service.csdnimg.cn/img_convert/01d3b71a2a8a088cf7454dd01ef49dfc.png)

- 每个接口做的事情非常明确，比如 Serializable，只负责序列化，Cloneable 只负责拷贝，Map 只负责定义 Map 的接口，整个图看起来虽然接口众多，但职责都很清晰；
- 复杂功能通过接口的继承来实现，比如 ArrayList 通过实现了 Serializable、Cloneable、RandomAccess、AbstractList、List 等接口，从而拥有了序列化、拷贝、对数组各种操作定义等各种功能；
- 上述类图只能看见继承的关系，组合的关系还看不出来，比如说 Set 组合封装 Map 的底层能力等。

上述设计的最大好处是，每个接口能力职责单一，众多的接口变成了接口能力的积累，假设我们想再实现一个数据结构类，我们就可以从这些已有的能力接口中，挑选出能满足需求的能力接口，进行一些简单的组装，从而加快开发速度。

这种思想在平时的工作中也经常被使用，我们会把一些通用的代码块抽象出来，沉淀成代码块池，碰到不同的场景的时候，我们就从代码块池中，把我们需要的代码块提取出来，进行简单的编排和组装，从而实现我们需要的场景功能。

# 1.集合迭代
在讲具体迭代方式之前，先来看一下迭代的顶层接口：Iterable。在看具体源码前先明白两个问题：
1. Iterable有什么用呢？实现了Itrable接口就能进行迭代（遍历）操作。
2. 为什么我在ArrayList中没有看见它实现Iterable呢？从上面的类图中可以看到，集合的顶级接口Collection实现了Iterable，但这里注意一点，Map并没有实现Iterable（map的迭代第二部分再讲解）。
```java
public interface Iterable<T> {
    Iterator<T> iterator()
        
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
    
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```
从接口中我们看到遍历的手段有两种：
 1.  迭代器Iterator，可以实现多个Iterator进行多种方法的迭代（前序，后序...)
  2. forEach方法，一般由具体子类重写该方法

这里注意一点，Iterator 和 forEach 都需要要进行迭代的类自行实现。

## 1.1 方式一：迭代器Iterator
**常由内部类实现，iterator方法返回，一般用于要对集合进行删除的情景**

- 迭代器是一种设计模式，封装了对集合的遍历，使得不用了解集合的内部细节，就可以用同样的方式遍历不同的集合
- 迭代器不允许使用集合方法进行集合增删，但是可以对集合的元素操作（如set()），还可以使用迭代器的remove()
  - 不能使用集合的 put 和 remove 方法
  - 可用于集合元素属性的修改：set方法
  - remove操作：是Iterator的remove方法

这里特别注意一点，一定要在next()后使用，比如删除第一个元素，要先next然后才能remove
```java
public interface Iterator<E> {
   
 	// 每次next之前，先调用此方法探测是否迭代到终点
    boolean hasNext();  
    
 	// 返回当前迭代元素 ，同时，迭代游标后移
    E next();           
              
     /*删除最近一次已近迭代出出去的那个元素。
     只有当next执行完后，才能调用remove函数。
     比如你要删除第一个元素，不能直接调用 remove()   而要先next一下( );
     在没有先调用next 就调用remove方法是会抛出异常的。
     这个和MySQL中的ResultSet很类似
    */
    void remove() 
    {
        throw new UnsupportedOperationException("remove");
    }
}
```
- 迭代器使用示例：
```java
// iterator是集合的自己Iterator构造方法
Iterator it = list.iterator();
while(it.hasNext) {
    it.next();
}
```
## 1.2 方式二：forEach方法
**本质是对for循环的封装，配合lamada使用，一般用于修改集合对象属性的情景**
```java
// ArrayList.forEach()
@Override
public void forEach(Consumer<? super E> action) {
  // 判断非空
  Objects.requireNonNull(action);
  // modCount的原始值被拷贝
  final int expectedModCount = modCount;
  final E[] elementData = (E[]) this.elementData;
  final int size = this.size;
  // 每次循环都会判断数组有没有被修改，一旦被修改，停止循环
  for (int i=0; modCount == expectedModCount && i < size; i++) {
    // 执行循环内容，action 代表我们要干的事情
    action.accept(elementData[i]);
  }
  // 数组如果被修改了，抛异常
  if (modCount != expectedModCount) {
    throw new ConcurrentModificationException();
  }
}
```
- 因为在Iterable接口中**default修饰**，所以必须**自身实现而子类不一定要重写**，在jdk8时所有集合都实现了forEach方法
- forEach：
  - 修改对象属性：通过set方法
  - 不能进行增删操作：modCount
- 使用示例：
```java
list.forEach(l -> {
    l.setName("zs");
    l.setAge(18);
})
```
## 1.3 方式三：增强for循环

**本质是对Iterator的简化与封装，一般用于只遍历集合的情况**

- 增强for循环：
  - 可以用于集合中元素属性值的修改：set方法
  - 但不能对集合新增或者删除：modCount控制
- 使用示例：
```java
public static void main(String[] args) {
        ArrayList<Person> people = new ArrayList<>();
        people.add(new Person("zs")); 
        people.add(new Person("lisi"));
    	
    	// 调用增强for循环
        for(Person p: people) {
            // 通过set修改对象属性值，成功
            p.setName("abc");
        }
        System.out.println(people); // abc,abc
}
```
# 2.Map迭代
从文章首部的类图中可以看到，map并未实现Itrable接口，那map该如何进行迭代呢？

1. 通过set的Iterator进行迭代

2. 最高层Map接口定义了forEach方法

## 2.1 方式一：Set.iterator
Map 对 key、value 和 entity（节点） 都提供了以 Set 为基础的迭代器。这些迭代器可以通过`map.~Set().iterator()`进行获取

- 迭代key：HashMap --keySet()--> KeySet --iterator()--> **KeyIterator**
- 迭代value：HashMap --values()--> Values--iterator()--> **ValueIterator**
- 迭代key：HashMap --entrySet()--> EntrySet--iterator()--> **EntryIterator**

虽然是不同的迭代器，但是它们本质上却没有区别，主要体现在以下两点：

1. 都继承了HashIterator，即拥有HashIterator的所有方法
	
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
使用示例：
```java
Iterator<Map.Entry<String,String>> it = map.entrySet().iterator
while (it.hasNext()) {
	Map.Entry<String,String>  me = it.next();
	// 获取key
	me.getkey();
	// 获取value
	me.getValue();
}
```
## 2.2 方式二：Map.forEach
Map中定义的forEach，default修饰，实际上也是调用entrySet
```java
default void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
            action.accept(k, v);
        }
}

```
HashMap的forEach方法源码：
```java
	@Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key, e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
       }
 }
```
# 3.批量操作

## 3.1 批量新增
下面列出 ArrayList.addAll 方法的源码：
```java
public boolean addAll(Collection<? extends E> c) {
  Object[] a = c.toArray();
  int numNew = a.length;
  // 确保容量充足，整个过程只会扩容一次
  ensureCapacityInternal(size + numNew); 
  // 进行数组的拷贝
  System.arraycopy(a, 0, elementData, size, numNew);
  size += numNew;
  return numNew != 0;
}
```
我们可以看到，整个批量新增的过程中，只扩容了一次，HashMap 的 putAll 方法也是如此，整个新增过程只会扩容一次，大大缩短了批量新增的时间，提高了性能。

所以当碰到集合批量拷贝，批量新增场景，要提高新增性能的时候 ，就可以从目标集合初始化方面入手。

这里也提醒了我们，在容器初始化的时候，最好能给容器赋上初始值，这样可以防止在 put 的过程中不断的扩容，从而缩短时间，上章 [HashSet](https://blog.csdn.net/weixin_43935927/article/details/108519853) 的源码演示了给 HashMap 赋初始值的公式为：取括号内两者的最大值（期望的值/0.75+1，默认值 16）。

使用示例：

在 List 和 Map 大量数据新增的时候，我们不要使用 for 循环 + add/put 方法新增，这样子会有很大的扩容成本，我们应该尽量使用 addAll 和 putAll 方法进行新增，下面以 ArrayList 为例写了一个 demo 如下，演示了两种方案的性能对比：
```java
@Test
public void testBatchInsert(){
  // 准备拷贝数据
  ArrayList<Integer> list = new ArrayList<>();
  for(int i=0;i<3000000;i++){
    list.add(i);
  }

  // for 循环 + add
  ArrayList<Integer> list2 = new ArrayList<>();
  long start1 = System.currentTimeMillis();
  for(int i=0;i<list.size();i++){
    list2.add(list.get(i));
  }
  log.info("单个 for 循环新增 300 w 个，耗时{}",System.currentTimeMillis()-start1);

  // 批量新增
  ArrayList<Integer> list3 = new ArrayList<>();
  long start2 = System.currentTimeMillis();
  list3.addAll(list);
  log.info("批量新增 300 w 个，耗时{}",System.currentTimeMillis()-start2);
}
```
最后打印出来的日志为：
```
16:52:59.865 [main] INFO demo.one.ArrayListDemo - 单个 for 循环新增 300 w 个，耗时1518

16:52:59.880 [main] INFO demo.one.ArrayListDemo - 批量新增 300 w 个，耗时8
```

可以看到，批量新增方法性能是单个新增方法性能的 189 倍，主要原因在于批量新增，只会扩容一次，大大缩短了运行时间，而单个新增，每次到达扩容阀值时，都会进行扩容，在整个过程中就会不断的扩容，浪费了很多时间

## 3.2 批量删除

批量删除 ArrayList 提供了 removeAll 的方法，HashMap 没有提供批量删除的方法，我们一起来看下 removeAll 的源码实现，是如何提高性能的：
```java
// 批量删除，removeAll 方法底层调用的是 batchRemove 方法
// complement 参数默认是 false,false 的意思是数组中不包含 c 中数据的节点往头移动
// true 意思是数组中包含 c 中数据的节点往头移动，这个是根据你要删除数据和原数组大小的比例来决定的
// 如果你要删除的数据很多，选择 false 性能更好，当然 removeAll 方法默认就是 false。
private boolean batchRemove(Collection<?> c, boolean complement) {
  final Object[] elementData = this.elementData;
  // r 表示当前循环的位置、w 位置之前都是不需要被删除的数据，w 位置之后都是需要被删除的数据
  int r = 0, w = 0;
  boolean modified = false;
    
  try {
    // 从 0 位置开始判断，当前数组中元素是不是要被删除的元素，不是的话移到数组头
    for (; r < size; r++)
      if (c.contains(elementData[r]) == complement)
        elementData[w++] = elementData[r];
  } finally {
    // r 和 size 不等，说明在 try 过程中发生了异常，在 r 处断开
    // 把 r 位置之后的数组移动到 w 位置之后(r 位置之后的数组数据都是没有判断过的数据，这样不会影响没有判断
    //  的数据，判断过的数据可以被删除)
    if (r != size) {
      System.arraycopy(elementData, r,
                       elementData, w,
                       size - r);
      w += size - r;
    }
      
    // w != size 说明数组中是有数据需要被删除的
    // 如果 w、size 相等，说明没有数据需要被删除
    if (w != size) {
      // w 之后都是需要删除的数据，赋值为空，帮助 gc。
      for (int i = w; i < size; i++)
        elementData[i] = null;
      modCount += size - w;
      size = w;
      modified = true;
    }
  }
  return modified;
}
```
我们看到 ArrayList 在批量删除时，如果程序执行正常，只有一次 for 循环，如果程序执行异常，才会加一次拷贝，而单个 remove 方法，每次执行的时候都会进行数组的拷贝（当删除的元素正好是数组最后一个元素时除外），当数组越大，需要删除的数据越多时，批量删除的性能会越差，所以在 ArrayList 批量删除时，强烈建议使用 removeAll 方法进行删除。

# 4.线程安全问题

我们说集合都是非线程安全的，这里说的非线程安全指的是集合类作为共享变量，被多线程读写的时候，才是不安全的，如果要实现线程安全的集合，在类注释中，JDK 统一推荐我们使用 Collections.synchronized* 类， Collections 帮我们实现了 List、Set、Map 对应的线程安全的方法， 如下图：
![图片描述](https://img-service.csdnimg.cn/img_convert/cf6989604600810fccbf0aed28d38600.png)
图中实现了各种集合类型的线程安全的方法，我们以 synchronizedList 为例，从源码上来看下，Collections 是如何实现线程安全的：

```java
// mutex 就是我们需要锁住的对象
final Object mutex;  

// 这些synchronized~~都是Collections的静态内部类
static class SynchronizedList<E> extends SynchronizedCollection<E> implements List<E> {
    
        private static final long serialVersionUID = -7754090372962971524L;
        // 通过组合的方式，传入需要保证线程安全的类（List）
    	// Collection.synchronizedList（list）
        final List<E> list;
        SynchronizedList(List<E> list, Object mutex) {
            super(list, mutex);
            this.list = list;
        }
        
	   // 我们可以看到，List 的所有操作都使用了 synchronized 关键字，来进行加锁
	   // synchronized 是一种悲观锁，能够保证同一时刻，只能有一个线程能够获得锁
        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        public E set(int index, E element) {
            synchronized (mutex) {return list.set(index, element);}
        }
        public void add(int index, E element) {
            synchronized (mutex) {list.add(index, element);}
        }
…………
}      
```

从源码中我们可以看到 Collections 是通过 synchronized 关键字给 List 操作数组的方法加上锁，来实现线程安全的。

# 5.两点注意
在文章的最后，再提出使用集合时的两点注意：
1. 重写equals & hashcode
	当集合的元素是自定义类时，自定义类强制实现 equals 和 hashCode 方法，并且两个都要实现。因为在集合中，除了 TreeMap 和 TreeSet 是通过比较器比较元素大小外，其余的集合类在判断索引位置和相等时，都会使用到 equals 和 hashCode 方法，这个在之前的源码解析中，我们有说到，所以当集合的元素是自定义类时，我们强烈建议覆写 equals 和 hashCode 方法，我们可以直接使用 IDEA 工具覆写这两个方法，非常方便

2. 迭代删除
所有集合类，在 for 循环进行删除时，如果直接使用集合类的 remove 方法进行删除，都会快速失败，报 ConcurrentModificationException 的错误，所以在任意循环删除的场景下，都建议使用迭代器进行删除；


