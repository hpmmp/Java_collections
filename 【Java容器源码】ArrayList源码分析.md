# 1.结构

ArrayList继承关系，核心成员变量，主要构造函数：
```java
public class ArrayList<E> extends AbstractList<E>
    	implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    	
    	//默认数组大小10
        private static final int DEFAULT_CAPACITY = 10;
    
        //数组存放的容器
        private static final Object[] EMPTY_ELEMENTDATA = {};
    
        //数组使用的大小，注：length是整个数组的大小
        private int size;
          
        //空数组，用于空参构造
        private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    		
    	//真正保存数据的数组，注：这里是Object类型 ===> 构造时传入泛型是必要的
      	transient  Object[] elementData;
    	
    	//---------------------------------------------------------------------
        
        //无参数直接初始化，数组大小为空
        //注：ArrayList 无参构造器初始化时，默认大小是空数组，并不是大家常说的 10，
        //   10 是在第一次 add 的时候扩容的数组值
        public ArrayList() {
            this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
        }
        
        //指定容量初始化
        public ArrayList(int initialCapacity) {
            if (initialCapacity > 0) {
              this.elementData = new Object[initialCapacity];
            } else if (initialCapacity == 0) {
              this.elementData = EMPTY_ELEMENTDATA;
            } else {
              throw new IllegalArgumentException("Illegal Capacity: "+
                                                 initialCapacity);
            }
        }
    
        //指定初始数据初始化
        // <? extends E>：类型，E及E的子类们
        // Collection<? extends E>：E及E的子类的集合
        public ArrayList(Collection<? extends E> c) {
            //elementData 是保存数组的容器，默认为 null
            elementData = c.toArray();
            //如果给定的集合（c）数据有值
            // size
            if ((size = elementData.length) != 0) {
                // c.toArray might (incorrectly) not return Object[] (see 6260652)
                //如果集合元素类型不是 Object 类型，我们会转成 Object
                if (elementData.getClass() != Object[].class) {
                    elementData = Arrays.copyOf(elementData, size, Object[].class);
                }
            } else {
                // 给定集合（c）无值，则默认空数组
                this.elementData = EMPTY_ELEMENTDATA;
            }
        }
    }
```

# 2.方法解析&api

## 2.1 增加
### add()
增加单个元素到容器中
```java
 public boolean add(E e) {
        //确保数组大小足够，不够需要扩容（期望容量=size+1）
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //直接赋值，线程不安全的
        //注：这里没有非空判断，因此可以加入null
        elementData[size++] = e;
    	// 这里虽然是boolean，但一般只会返回true
        return true;
    }

```
### addAll()
批量增加，即增加多个元素（集合）到容器中
```java
	public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        // 确保容量充足，整个过程只会扩容一次（期望容量=size+a.length)
        ensureCapacityInternal(size + numNew);  // Increments modCount
        // 直接将要加入的集合拷贝到elementData后面即可
        // 注：Arrays.copyOf适用于1-->2的拷贝,而sys..适用于对原数组或指定数组的操作
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        // 只有要增加的集合为空时，返回false
        return numNew != 0;
      }
```
    
## 2.2 扩容
### ensureCapacityInternal()
计算期望的最小容量
```java
    private void ensureCapacityInternal(int minCapacity) {
      // 只有当elementData为空（即构造时没有传入容量 && 第一次扩容)，才使用默认大小10
      if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
      }
      // 判断是否需要扩容
      ensureExplicitCapacity(minCapacity);
    }
```
### ensureExplicitCapacity()
判断是否需要扩容，并修改modCount记录数组变化。这里需要明白一点，该方法没必要返回bool值，因为不能因为容量不够就不放
```java
    private void ensureExplicitCapacity(int minCapacity) {
      // 记录数组被修改
      modCount++;
      // 如果我们期望的最小容量大于目前数组的长度，那么就扩容
      // 注：这里当minCapacity=length时也不扩容
      if (minCapacity - elementData.length > 0)
        grow(minCapacity);
    }
```
### grow()
执行扩容，因为数组在创建时大小就确定了，所以所谓的扩容并不是将当前数组变大了，而是创建一个新的大数组，然后将原来数组元素拷贝过去，最后再将elementData指针指向这个新数组。
```java
    private void grow(int minCapacity) {
      int oldCapacity = elementData.length;
      // newCapacity = 1.5 oldCapacity
      int newCapacity = oldCapacity + (oldCapacity >> 1);
    
      // 如果扩容后的值 < 我们的期望值，就以期望值进行扩容
      if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    
      // 如果扩容后的值 > jvm 所能分配的数组的最大值，那么就用 Integer 的最大值
      if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
     
      // 通过复制进行扩容
      elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
我们需要注意的四点是：

1. 新增时，并没有对值进行严格的校验，所以 ArrayList 是允许 null 值的。
2. 扩容的规则并不是翻倍，是原来容量大小 + 容量大小的一半，即原来容量的 1.5 倍。
3. ArrayList 中的数组的最大值是 Integer.MAX_VALUE，超过这个值，JVM 就不会给数组分配内存空间了。
   源码在扩容的时候，有数组大小溢出意识，就是说扩容后数组下界不能小于 0，上界不能大于 Integer 的最大值
4. 扩容完成之后，赋值是非常简单的，直接往数组上添加元素即可：elementData [size++] = e。也正是通过这种简单赋值，没有任何锁控制，所以这里的操作是线程不安全的

**扩容的本质**

扩容是通过这行代码来实现的：Arrays.copyOf(elementData, newCapacity);，这行代码描述的本质是数组之间的拷贝，扩容是会先新建一个符合我们预期容量的新数组，然后把老数组的数据拷贝过去，我们通过 System.arraycopy 方法进行拷贝，此方法是 native 的方法，源码如下：
```java
    /**
     * @param src     被拷贝的数组
     * @param srcPos  从数组那里开始
     * @param dest    目标数组
     * @param destPos 从目标数组那个索引位置开始拷贝
     * @param length  拷贝的长度 
     * 此方法是没有返回值的，通过 dest 的引用进行传值
     */
    public static native void arraycopy(Object src, int srcPos,
                                        Object dest, int destPos,
                                        int length);
```
我们可以通过下面这行代码进行调用，newElementData 表示新的数组：
```java
    System.arraycopy(elementData, 0, newElementData, 0,Math.min(elementData.length,newCapac
```
## 2.3  删除
### remove()
寻找要删除元素的索引
```java
    public boolean remove(Object o) {
      // 如果要删除的值是 null，找到第一个值是 null 的删除
      // 注：这里把null单独出来，是因为e[idx]==o, 而非空是e[idx].equals(o)
      if (o == null) {
        for (int index = 0; index < size; index++)
          if (elementData[index] == null) {
            fastRemove(index);
            return true;
          }
      } else {
        // 如果要删除的值不为 null，找到第一个和要删除的值相等的删除
        for (int index = 0; index < size; index++)
          // 这里是根据  equals 来判断值相等的，相等后再根据索引位置进行删除
          if (o.equals(elementData[index])) {
            fastRemove(index);
            return true;
          }
      }
      return false;
    }
```
我们需要注意的两点是：

- 新增的时候是没有对 null 进行校验的，所以删除的时候也是允许删除 null 值的
- 找到值在数组中的索引位置，是通过 equals 来判断的，如果数组元素不是基本类型（Integer，String等），而是自定义类型（如User，Item等），则需要重写equals方法

### fastRemove()
执行删除，即数组拷贝
```java
    private void fastRemove(int index) {
      // 记录数组的结构要发生变动了
      modCount++;
      // numMoved 表示删除 index 位置的元素后，需要从 index 后移动多少个元素到前面去
      // 减 1 的原因，是因为 size 从 1 开始算起，index 从 0开始算起
      int numMoved = size - index - 1;
      if (numMoved > 0)
        // 从 index +1 位置开始被拷贝，拷贝的起始位置是 index，长度是 numMoved
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
      //数组最后一个位置赋值 null，帮助 GC
      elementData[--size] = null;
    }
```
### removeAll()
批量删除
```java
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
      }
```
### batchRemove()
ArrayList 在批量删除时，如果程序执行正常，只有一次 for 循环，如果程序执行异常，才会加一次拷贝
```java
    // complement 参数默认是 false,false 的意思是数组中不包含 c 中数据的节点往头移动
    // true 意思是数组中包含 c 中数据的节点往头移动，这个是根据你要删除数据和原数组大小的比例来决定的
    // 如果你要删除的数据很多，选择 false 性能更好，当然 removeAll 方法默认就是 false。
    private boolean batchRemove(Collection<?> c, boolean complement) {
      final Object[] elementData = this.elementData;
      // r 表示当前循环的位置、w 位置之前都是不需要被删除的数据，w 位置之后都是需要被删除的数据
      int r = 0, w = 0;
      boolean modified = false;
      
        // 双指针执行删除
        try {
        // 从 0 位置开始判断，当前数组中元素是不是要被删除的元素，不是的话移到数组头
        for (; r < size; r++)
          if (c.contains(elementData[r]) == complement)
            elementData[w++] = elementData[r];
      	} finally {
        // r 和 size 不等，说明在 try 过程中发生了异常，在 r 处断开
        // 把 r 位置之后的数组移动到 w 位置之后(r 位置之后的数组数据都是没有判断过的数据，
        // 这样不会影响没有判断的数据，判断过的数据可以被删除）
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
## 2.4 修改
### set()
修改指定索引的元素
```java
    public E set(int index, E element) {
        rangeCheck(index);
    
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```
## 2.5 迭代器
### iterator()
iterator 方法的作用是给用户返回迭代器
```java
    public Iterator<E> iterator() {
        return new Itr();
    }

    /**
    *Itr是对迭代器的实现
    */
    private class Itr implements Iterator<E>{
        
        // 迭代过程中，下一个元素的位置，默认从 0 开始。
        int cursor;
        
    	// 记录当前元素索引，因为next获取到元素后cursor++
        // 单独出来的意义还在于多线程下防止重复删除，因为删除后
        int lastRet = -1; 
    	
        // expectedModCount 表示迭代过程中，期望的版本号；modCount 表示数组实际的版本号
        int expectedModCount = modCount;
        
        //...
    }
```
- modCount：保证在当前迭代中，不在对集合进行增加删除操作，add/remove均会改变modCount
- expectedModCount：记录在迭代开始前的modCount，在迭代过程中若出现变化则抛异常

### hasNext()
判断是否还有下一个被迭代元素，即是否已经到数组尾部了（index=length-1）
```java
    public boolean hasNext() {
           // cursor 表示下一个元素的位置，size 表示实际大小，如果两者相等，说明已经没有元素可以迭代了，	    
           // 如果不等，说明还可以迭代
      	   return cursor != size;
    }
```
### next()
返回当前元素，并为下一次迭代做准备（cursor+1）。这里注意一点，如果在迭代时，数组被修改了，那迭代就出错了，所以迭代时原则上不允许增删(可以修改set)
```java
    public E next() {
           //迭代过程中，判断版本号有无被修改，有被修改，抛 ConcurrentModificationException 异常
           // 注：增、删都会引起modCount改变，但修改（set）不会
           checkForComodification();
           //本次迭代过程中，元素的索引位置
           int i = cursor;
           if (i >= size)
              throw new NoSuchElementException();
           // 注：ArrayList.this.~可以拿到外部类的属性
           Object[] elementData = ArrayList.this.elementData;
           if (i >= elementData.length)
              throw new ConcurrentModificationException();
            // 下一次迭代时，元素的位置，为下一次迭代做准备
            cursor = i + 1;
            // 返回元素值
            return (E) elementData[lastRet = i];
        }
        
        // 版本号比较
        final void checkForComodification() {
          if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        }
```
### remove()
提供一个迭代时，可以删除当前元素的方法
    */
```java
    public void remove() {
          // 如果上一次操作时，数组的位置已经小于 0 了，说明数组已经被删除完了
          if (lastRet < 0)
            throw new IllegalStateException();
          //迭代过程中，判断版本号有无被修改，有被修改，抛 ConcurrentModificationException 异常
          checkForComodification();
    
          try {
            // 调用ArrayList的删除方法，即modCount也会++
            ArrayList.this.remove(lastRet);
            // 更新cursor，其实也就是回退一位
            cursor = lastRet;
            // -1 表示元素已经被删除，这里也防止重复删除
            lastRet = -1;
            // 删除元素时 modCount 的值已经发生变化，在此赋值给 expectedModCount，保证下次迭代时一致
            expectedModCount = modCount;
          } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
          }
     }
```
## 2.6 toArray()
常用于将List转为数组
```java
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }
```
在ArrayList中还有一个方法：toArray(T[])。但该方法需注意size与length的关系，注意避免出现错误
```java
    public <T> T[] toArray(T[] a) {
      // 如果数组长度不够，按照 List 的大小进行拷贝，return 的时候返回的都是正确的数组
      if (a.length < size)
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        
      // 长度刚好则对传入数组进行拷贝
      System.arraycopy(elementData, 0, a, 0, size);
        
      // 数组长度大于 List 大小的，赋值为 null（其后空间GC）
      if (a.length > size)
        a[size] = null;
      return a;
    }
```
