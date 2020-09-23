## 1.说说对 ArrayList 的理解？

 ArrayList 内容很多，可以先从总体架构入手，然后再以某个细节作为突破口，比如这样：ArrayList 底层数据结构是个数组，其 API 都做了一层对数组底层访问的封装，比如说 add 方法的过程是……（这里具体可以参考 [ArrayList 源码分析](https://blog.csdn.net/weixin_43935927/article/details/108466664)中 add 的过程）。


另外，对 LinkedList 的理解也是同样套路。


## 2.扩容相关问题

### 2.1 ArrayList 无参数构造器构造，现在 add 一个值进去，此时数组的大小是多少，下一次扩容前最大可用大小是多少？

答：此处数组的大小是 1，下一次扩容前最大可用大小是 10，因为 ArrayList 第一次扩容时，是有默认值的，默认值是 10，在第一次 add 一个值进去时，数组的可用大小被扩容到 10 了。

### 2.2 如果连续往 list 里面新增值，增加到第 11 个的时候，数组的大小是多少？

答：这里实际问的是扩容的公式，当增加到 11 的时候，此时我们希望数组的大小为 11，但实际上数组的最大容量只有 10，不够了就需要扩容，扩容的公式是：oldCapacity + (oldCapacity>> 1)，oldCapacity 表示数组现有大小，目前场景计算公式是：10 + 10 ／2 = 15，然后我们发现 15 已经够用了，所以数组的大小会被扩容到 15。

### 2.3 数组初始化，被加入一个值后，如果使用 addAll 方法，一下子加入 15 个值，那么最终数组的大小是多少？

答：在上一题中已经计算出来数组在加入一个值后，实际大小是 1，最大可用大小是 10 ，现在需要一下子加入 15 个值，那我们期望数组的大小值就是 16，此时数组最大可用大小只有 10，明显不够，需要扩容，扩容后的大小是：10 + 10 ／2 = 15，这时候发现扩容后的大小仍然不到我们期望的值 16，这时候源码中有一种策略如下：

```java
// newCapacity 本次扩容的大小，minCapacity 我们期望的数组最小大小
// 如果扩容后的值 < 我们的期望值，我们的期望值就等于本次扩容的大小
if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
```
所以最终数组扩容后的大小为 16。具体源码请参考[ArrayList 源码分析](https://blog.csdn.net/weixin_43935927/article/details/108466664)的grow方法。

### 2.4 现在有一个很大的数组需要拷贝，原数组大小是 5k，请问如何快速拷贝？

答：因为原数组比较大，如果新建新数组的时候，不指定数组大小的话，就会频繁扩容，频繁扩容就会有大量拷贝的工作，造成拷贝的性能低下，所以在新建数组时，指定新数组的大小为 5k 即可。

### 2.5 为什么说扩容会消耗性能？
答：扩容底层使用的是 System.arraycopy 方法，会把原数组的数据全部拷贝到新数组上，所以性能消耗比较严重。

### 2.6 源码扩容过程有什么值得借鉴的地方？
答：有两点：

- 扩容的思想值得学习，通过自动扩容的方式，让使用者不用关心底层数据结构的变化，封装得很好，1.5 倍的扩容速度，可以让扩容速度在前期缓慢上升，在后期增速较快，大部分工作中要求数组的值并不是很大，所以前期增长缓慢有利于节省资源，在后期增速较快时，也可快速扩容。
- 扩容过程中，有数组大小溢出的意识，比如要求扩容后的数组大小，不能小于 0，不能大于 Integer 的最大值。

## 3. 删除相关问题
### 3.1 有一个 ArrayList，数据是 2、3、3、3、4，中间有三个 3，现在通过 for (int i=0;i<list.size ();i++) 的方式，想把值是 3 的元素删除，请问可以删除干净么？最终删除的结果是什么，为什么？删除代码如下：
```java
List<String> list = new ArrayList<String>() {{
  add("2");
  add("3");
  add("3");
  add("3");
  add("4");
}};
for (int i = 0; i < list.size(); i++) {
  if (list.get(i).equals("3")) {
    list.remove(i);
  }
}
```

答：不能删除干净，最终删除的结果是 2、3、4，有一个 3 删除不掉，原因我们看下图：
![图片描述](https://img-blog.csdnimg.cn/img_convert/a469009fd68afd59341f113236b95b5f.png)从图中我们可以看到，每次删除一个元素后，该元素后面的元素就会往前移动，而此时循环的 i 在不断地增长，最终会使每次删除 3 的后一个 3 被遗漏，导致删除不掉。

### 3.2 还是上面的 ArrayList 数组，我们通过增强 for 循环进行删除，可以么？

答：不可以，会报错。因为增强 for 循环过程其实调用的就是迭代器的 next () 方法，当你调用 list.remove () 方法进行删除时，modCount 的值会 +1，而这时候迭代器中的 expectedModCount 的值却没有变，导致在迭代器下次执行 next () 方法时，expectedModCount != modCount 就会报 ConcurrentModificationException 的错误。

### 3.3 还是上面的数组，如果删除时使用 Iterator.remove () 方法可以删除么，为什么？

答：可以的，因为 Iterator.remove () 方法在执行的过程中，会把最新的 modCount 赋值给 expectedModCount，这样在下次循环过程中，modCount 和 expectedModCount 两者就会相等。

### 3.4 以上三个问题对于 LinkedList 也是同样的结果么？

答：是的，虽然 LinkedList 底层结构是双向链表，但对于上述三个问题，结果和 ArrayList 是一致的。

## 4.与LinkedList对比的问题
### 4.1  ArrayList 和 LinkedList 有何不同？
答：可以先从底层数据结构开始说起，然后以某一个方法为突破口深入，比如：
* 最大的不同是两者底层的数据结构不同，ArrayList 底层是数组，LinkedList 底层是双向链表
* 两者的数据结构不同也导致了操作的 API 实现有所差异，拿新增实现来说，ArrayList 会先计算并决定是否扩容，然后把新增的数据直接赋值到数组上，而 LinkedList 仅仅只需要改变插入节点和其前后节点的指向位置关系即可。

### 4.2 ArrayList 和 LinkedList 应用场景有何不同？
答：
* ArrayList 更适合于快速的查找匹配，不适合频繁新增删除，像工作中经常会对元素进行匹配查询的场景比较合适
* LinkedList 更适合于经常新增和删除，对查询反而很少的场景。

### 4.3 ArrayList 和 LinkedList 两者有没有最大容量？

答：
* ArrayList 有最大容量的，为 Integer 的最大值，大于这个值 JVM 是不会为数组分配内存空间的
* LinkedList 底层是双向链表，理论上可以无限大。但源码中，LinkedList 实际大小（size）用的是 int 类型，这也说明了 LinkedList 不能超过 Integer 的最大值，不然会溢出。

### 4.4 ArrayList 和 LinkedList 是如何对 null 值进行处理的？

答：
* ArrayList 允许 null 值新增，也允许 null 值删除。删除 null 值时，是从头开始，找到第一值是 null 的元素删除
* LinkedList 新增删除时对 null 值没有特殊校验，是允许新增和删除的。

## 5.线程安全问题
### 5.1 ArrayList 和 LinedList 是线程安全的么，为什么？
答：
* 当两者作为非共享变量时，比如说仅仅是在方法里面的局部变量时，是没有线程安全问题的，只有当两者是共享变量时，才会有线程安全问题。
* 主要的问题点在于多线程环境下，所有线程任何时刻都可对数组和链表进行操作，这会导致值被覆盖，甚至混乱的情况。就像ArrayList 自身的 elementData、size、modConut 在进行各种操作时，都没有加锁，而且这些变量的类型并非是可见（volatile）的，所以如果多个线程对这些变量进行操作时，可能会有值被覆盖的情况。

如果有线程安全问题，在迭代的过程中，会频繁报 ConcurrentModificationException 的错误，意思是在我当前循环的过程中，数组或链表的结构被其它线程修改了

### 5.2 如何解决线程安全问题？
答：Java 源码中推荐使用 Collections#synchronizedList 进行解决，Collections#synchronizedList 的返回值是 List 的每个方法都加了 synchronized 锁，保证了在同一时刻，数组和链表只会被一个线程所修改，但是性能大大降低，具体实现源码：
```java
public boolean add(E e) {
    synchronized (mutex) {// synchronized 是一种轻量锁，mutex 表示一个当前 SynchronizedList
        return c.add(e);
    }
}
```
另外，还可以采用 JUC 的 CopyOnWriteArrayList 并发 List 来解决。
