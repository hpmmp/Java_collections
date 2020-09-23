# 1.结构

LinkedList 继承关系，核心成员变量，主要构造函数：
```java
    public class LinkedList<E>
        extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
     	
     	// Node，双向链表
        private static class Node<E> {
            E item;// 节点值
            Node<E> next; // 指向的下一个节点
            Node<E> prev; // 指向的前一个节点
    
            // 初始化参数顺序分别是：前一个节点、本身节点值、后一个节点
            Node(Node<E> prev, E element, Node<E> next) {
                this.item = element;
                this.next = next;
                this.prev = prev;
            }
    	}
    	
    	//------------------------成员变量-------------------------------------
    	
    	transient int size = 0;
    	
    	// 记录头结点，它的前一个结点=null
        transient Node<E> first;
    	
    	// 记录尾结点，它的后一个结点=null
    	// 当 first = last = null时表示链表为空
    	// 当 first = last != null时表示只有一个节点
        transient Node<E> last;
        
        //--------------------------构造方法-------------------------------------
        
        public LinkedList() {
        }
        
         public LinkedList(Collection<? extends E> c) {
            this();
            addAll(c);
        }
        
        // ........
    }
```
# 2.方法解析&api

## 2.1 尾插

追加节点时，我们可以选择追加到链表头部，还是追加到链表尾部，add 方法默认是从尾部开始追加，addFirst 方法是从头部开始追加，我们分别来看下两种不同的追加方式：

### **add()**
```java
    public boolean add(E e) {
            linkLast(e);
            return true;
    }
```
### **linkLast()**

尾插的核心逻辑如下：
* newNode.pre = last
* last.next = newNode    （注：考虑last=null情况（链表为空，这时仅更新头结点即可））
 * last = newNode
```java
    void linkLast(E e) {
        // 把尾节点数据暂存，为last.next做准备，其实改变一下顺序就可以不要这个l了
        final Node<E> l = last;
        
        final Node<E> newNode = new Node<>(l, e, null); // 1
        last = newNode; // 2
    	
        // 空链表，l=null,l.next报空指针
        if (l == null)
            first = newNode;
        else
            l.next = newNode; // 3
        
        // size和版本更改
        size++;
        modCount++;
    }
```
## 2.2 头插
要对LinkedList头插时是调用addFirst方法
### **addFirst()**
```java
    public void addFirst(E e) {
            linkFirst(e);
    }
```
### **linkFirst()**
   头插核心逻辑如下：
   * newNode.next = first;
    * first.prev = newNode;  （注：考虑first=null（链表为空，只用更新last即可））
    * first = newNode;
```java
    private void linkFirst(E e) {
        // 头节点赋值给临时变量
        final Node<E> f = first;
        
        final Node<E> newNode = new Node<>(null, e, f); // 1
    
        first = newNode;  // 2
        
        // 链表为空，f=null, f.prev报空指针
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;  // 3
        
        // 更新size和版本号
        size++;
        modCount++;
    }
```
## 2.3 删除指定元素

节点删除的方式和追加类似，我们可以删除指定元素，或者从头部（尾部）删除，删除操作会把节点的值，前后指向节点都置为 null，帮助 GC 进行回收

### **remove()**

删除指定元素；该方法在此处的作用是找到要删除的节点。注意，只有链表有这个节点且成功删除才返回true
```java
    public boolean remove(Object o) { 
            if (o == null) {
                for (Node<E> x = first; x != null; x = x.next) {
                    // null用 == 判断
                    if (x.item == null) {
                        unlink(x);
                        return true;
                    }
                }
            } else {
                for (Node<E> x = first; x != null; x = x.next) {
                    // 调用equals判断，若传入的类无equals需要重写
                    if (o.equals(x.item)) {
                        unlink(x);
                        return true;
                    }
                }
            }
            return false;  // 链表无要删除元素，或链表为空
    }
```
注：remove还可以根据索引删除
```java
    public E remove(int index) { 
            checkElementIndex(index); // 链表为空，抛出异常
            return unlink(node(index));
    }
```
### **unlink()**

删除的核心逻辑如下：
 * x.prev.next = x.next   （注：考虑x.prev=null(x是first，直接更新first））
*  x.next.prev = x.prev.prev   （注：考虑x.next=null(x是last，直接更新last））
```java
    E unlink(Node<E> x) {
            // assert x != null;
            final E element = x.item;
            final Node<E> next = x.next;
            final Node<E> prev = x.prev;
    		
         	// 如果prev=null,则当前节点为头结点
            if (prev == null) {
                // 直接将头结点赋成next
                first = next;
            } else {
                prev.next = next; // 1
                x.prev = null; // 帮助 GC 回收该节点
            }
    		
         	// 如果next=null，则当前节点为尾结点
            if (next == null) {
                last = prev;
            } else {
                next.prev = prev; // 2
                x.next = null; // 帮助 GC 回收该节点
            }
    		
            x.item = null; // 帮助 GC 回收该节点
         
         	// 修改size及版本
            size--;
            modCount++;
         
            return element;
        }
```
## 2.4 删除头节点
### **remove()**

删除头节点，队列为空时抛出异常。这里注意，与删除指定元素时需要传入一个参数，而删除头节点时为空参。
```java
    public E remove() {
            return removeFirst();
    }
```
### **removeFirst()**

判断当前链表时否为空
```java
    public E removeFirst() {
            final Node<E> f = first;
            if (f == null)
                throw new NoSuchElementException();
            return unlinkFirst(f);
     }
```
### **unLinkFirst()**

执行删除头节点，具体删除逻辑如下
  * first.next.pre = null;（注：考虑first=null（链表为空）， first.next=null（尾结点，即链表仅一个节点））
  * first = first.next;
```java
    private E unlinkFirst(Node<E> f) {
        
        final E element = f.item; // 拿出头节点的值，作为方法的返回值
        final Node<E> next = f.next; // 拿出头节点的下一个节点
        
        //帮助 GC 回收头节点
        f.item = null;
        f.next = null;
        
        first = next;  // 1
        
        // next为空表示链表只有一个节点
        if (next == null)
            last = null;
        else
            next.prev = null; // 2
        
        //修改链表大小和版本
        size--;
        modCount++;
        return element;
    }
```
从源码中我们可以了解到，链表结构的节点新增、删除都非常简单，仅仅把前后节点的指向修改下就好了，所以 LinkedList 新增和删除速度很快。

## 2.5 查询
链表查询某一个节点是比较慢的，需要挨个循环查找才行，我们看看 LinkedList 的源码是如何寻找节点的

### **get()**
根据索引进行查找
```java 
   public E get(int index) {
            checkElementIndex(index);
            return node(index).item;
    }
```
### **node()**
```java
    Node<E> node(int index) {
        // 如果 index 处于队列的前半部分，从头开始找，size >> 1 是 size 除以 2 的意思。
        if (index < (size >> 1)) {
            // 取头节点
            Node<E> x = first;
            // 直到 for 循环到 index 的前一个 node 停止
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {// 如果 index 处于队列的后半部分，从尾开始找
            // 取尾结点
            Node<E> x = last;
            // 直到 for 循环到 index 的后一个 node 停止
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
从源码中我们可以发现，LinkedList 并没有采用从头循环到尾的做法，而是采取了简单二分法，首先看看 index 是在链表的前半部分，还是后半部分。如果是前半部分，就从头开始寻找，反之亦然。通过这种方式，使循环的次数至少降低了一半，提高了查找的性能，这种思想值得我们借鉴

# 3.迭代器

因为 LinkedList 要实现双向的迭代访问，所以使用 Iterator 接口肯定不行了，因为 Iterator 只支持从头到尾的访问。Java 新增了一个迭代接口，叫做：ListIterator，这个接口提供了向前和向后的迭代方法，如下所示：
| 迭代顺序   | 方法   |
|--|--|
| 从尾到头迭代方法 | hasPrevious、previous、previousIndex |
| 从头到尾迭代方法	| hasNext、next、nextIndex  |                
**listIterator()**
返回迭代器，可以传入index，表示从指定节点开始迭代，可前可后
```java
    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }

    /**
    *ListItr,双向迭代器
    */
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;//上一次执行 next() 或者 previos() 方法时的节点位置
        private Node<E> next;//下一个节点
        private int nextIndex;//下一个节点的位置
        //expectedModCount：期望版本号；modCount：目前最新版本号
        private int expectedModCount = modCount;
        
        ListItr(int index) {
              // assert isPositionIndex(index);
              next = (index == size) ? null : node(index);
              nextIndex = index;
        }
    }
```
## 3.1 从前向后迭代
### **hasNext()**
判断还有没有下一个元素，还是通过index和size控制
```java
    public boolean hasNext() {
        return nextIndex < size;// 下一个节点的索引小于链表的大小，就有
    }
```
### **next()**
取下一个元素，并后移
```java
    public E next() {
        //检查期望版本号有无发生变化
        checkForComodification();
        if (!hasNext())//再次检查
            throw new NoSuchElementException();
        // next 是当前节点，在上一次执行 next() 方法时被赋值的。
        // 第一次执行时，是在初始化迭代器的时候，next 被赋值的
        lastReturned = next;
        // next 是下一个节点了，为下次迭代做准备
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }
```
## 3.2 从后向前迭代
### **hasPrevious()**

如果上次节点索引位置大于 0，就还有节点可以迭代
```java
    public boolean hasPrevious() {
        return nextIndex > 0;
    }
```
### **previous()**

```java
    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();
        // next 为空场景：1:说明是第一次迭代，取尾节点(last);2:上一次操作把尾节点删除掉了
        // next 不为空场景：说明已经发生过迭代了，直接取前一个节点即可(next.prev)
        lastReturned = next = (next == null) ? last : next.prev;
        // 索引位置变化
        nextIndex--;
        return lastReturned.item;
    }
```
## 3.3 删除：remove
迭代时，删除当前元素
```java
  public void remove() {
        checkForComodification();
        // lastReturned 是本次迭代需要删除的值，分以下空和非空两种情况：
        // lastReturned 为空，说明调用者没有主动执行过 next() 或者 previos()，直接报错
        // lastReturned 不为空，是在上次执行 next() 或者 previos()方法时赋的值
        if (lastReturned == null)
            throw new IllegalStateException();
        Node<E> lastNext = lastReturned.next;
        //删除当前节点
        unlink(lastReturned);
        // next == lastReturned 的场景分析：从尾到头递归顺序，并且是第一次迭代，并且要删除最后一个元素的情况
        // 这种情况下，previous()方法里面设置了 lastReturned=next=last,所以 next 和l astReturned 会相等
        if (next == lastReturned)
            // 这时候 lastReturned 是尾节点，lastNext 是 null，所以 next 也是 null，	
            // 这样在 previous() 执行时，发现 next 是 null，就会把尾节点赋值给 next
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }
```
# 4.Queue的实现
LinkedList 实现了 Queue 接口，在新增、删除、查询等方面增加了很多新的方法，这些方法在平时特别容易混淆，在链表为空的情况下，返回值也不太一样，下面列一个表格，方便大家记录：

| | 返回异常 | 	返回特殊值 | 底层实现 |
|--|--|--|--|
  新增 | 	add()   |	offer() | 	底层实现相同，offer直接调用add                           
  删除 | 	remove() 	|poll() |链表为空时，remove 会抛出异常，poll 返回 null
  查找 | 	element()	|peek()  |	链表为空时，element 会抛出异常，peek 返回 null

PS：Queue 接口注释建议 add 方法操作失败时抛出异常，但 LinkedList 实现的 add 方法一直返回 true。
LinkedList 也实现了 Deque 接口，对新增、删除和查找都提供从头开始，还是从尾开始两种方向的方法，比如 remove 方法，Deque 提供了 removeFirst 和 removeLast 两种方向的使用方式，但当链表为空时的表现都和 remove 方法一样，都会抛出异常。
