上一节分析了[JDK8中HashMap](https://blog.csdn.net/weixin_43935927/article/details/108501752)的结构和主要方法，这节就对比一下JDK7中的HashMap的实现
## 1.增加元素
首先来看看jdk7中HashMap增加元素的方法。
### put()
1. 根据key计算hash值，然后根据hash得到具体槽点位置
2. 遍历当前槽点的链表
   - 如果发现相同key直接覆盖，返回oldValue
   - 没有相同key的节点，就将新节点尾插（addEntry）
```java
public V put(K key, V value)
{
    ......
    // 计算Hash值
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    // 遍历当前槽点的链表
    // 注：jdk7中还没有红黑树，因此解决哈希碰撞就只有拉链
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        // 如果该key已存在，则替换掉旧的value
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    // 该key不存在，需要增加一个结点
    addEntry(hash, key, value, i);
    return null;
}
```
### addEntry()
当key不存在时，调用addEntry()方法添加新节点
```java
void addEntry(int hash, K key, V value, int bucketIndex)
{
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
    // 查看当前的size是否超过了阈值threshold，如果超过，需要resize
    if (size++ >= threshold)
        resize(2 * table.length); // 直接传入扩容后的大小，2*len
}
```
## 2.扩容逻辑
下面就到了核心部分，扩容。
### resize()
判断能否扩容，并创建新数组
```java
// 传入新的容量  
void resize(int newCapacity) {   
    // 引用扩容前的Entry数组  
    Entry[] oldTable = table;    
    int oldCapacity = oldTable.length;  
    // 扩容前的数组大小如果已经达到最大(2^30)了 
    if (oldCapacity == MAXIMUM_CAPACITY) {  
        //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了 
        threshold = Integer.MAX_VALUE;  
        return;  
    }  
    
	// 初始化一个新的Entry数组
    Entry[] newTable = new Entry[newCapacity];  
    //！！将数据转移到新的Entry数组里，这里包含最重要的重新定位
    transfer(newTable);    
    // 将当前线程创建的新数组置为Map的数组
    table = newTable; 
    // 修改阈值  
    threshold = (int) (newCapacity * loadFactor);
}
```
### transfer(真正扩容)
扩容策略：
1. 将原数组的所有结点重新计算位置
2. 保存原链表：e保存当前节点，next记录下一个结点（e.next)
   - 新槽点头插（注：put时是尾插）
     - 连接：e.next = newTab[i]
     - 下移：newTab[i] = e
   - 后移处理下一个结点：e = next
```java
void transfer(Entry[] newTable) {  
    // src引用了旧的Entry数组  
    Entry[] src = table;                    
    int newCapacity = newTable.length; 
    // 遍历旧的Entry数组  
    for (int j = 0; j < src.length; j++) { 
        Entry<K, V> e = src[j];             
        if (e != null) {  
            // 释放旧Entry数组的对象引用
            src[j] = null;  
            do {  
                // next用来保存当前节点之后的节点
                Entry<K, V> next = e.next;  
                //！！重新计算每个元素在数组中的位置 
                int i = indexFor(e.hash, newCapacity);  
                // 新槽点头插
                e.next = newTable[i]; //标记[1]  
                //将元素放在数组上  
                newTable[i] = e; 
                // 原链表节点后移
                e = next;              
            } while (e != null);  
        }  
    }  
}
```
## 3.扩容时死锁问题分析

- 产生死锁的前提
  - 多个线程同时都要扩容
    - 两个线程都已经在自己的栈空间建立了数组
    - 且一个线程刚好执行到：保存完e，与next，然后挂起
  - e与next在扩容后还在一个槽位
```java
do {
  Entry<K,V> next = e.next; // 线程一执行至此被挂起，执行线程二
  int i = indexFor(e.hash, newCapacity);
  e.next = newTable[i];
  newTable[i] = e;
  e = next;
} while (e != null);
```
- 死锁产生原因
  三次循环后成环，新槽点上的两个节点 A.next = B && B.next = A
- 过程图解
  - 线程一记录执行状态，挂起
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200909211532924.png#pic_center)
  - 线程二正常完成扩容，扩容后e next仍在同一个槽点，且逆序排列；线程一记录的e next不变
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200909211649846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)
  
  - 线程一恢复运行，第一次循环：先将e（A）插入到槽点（3）；然后e=next（B）
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200909211719835.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)

  - 线程一继续运行，第二次循环：next=e.next(A)，将e（B）头插到槽点（3），e=next（A）
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200909211757546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)
  
  - 线程一死锁产生，第三次循环：next=e.next=null，将e（A）再头插到槽点（3），形成环
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200909211827174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)

