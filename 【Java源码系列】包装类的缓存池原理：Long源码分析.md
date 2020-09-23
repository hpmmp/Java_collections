本文以Long为例，来看看包装类中缓存池设计的思路。
# 1.结构
Long 中缓存池相关代码：
```java
public final class Long extends Number implements Comparable<Long> {
    // 用来保存具体long值
    private final long value;
    
    // 在构造时传入要封装的long值
    public Long(long value) {
        this.value = value;
    }
    
    // 注：这里需要静态成员（缓存数组：cache）所以是静态内部类
    private static class LongCache {
        // 私有构造函数，不能被new出来
    	private LongCache(){}
    	// 缓存（本质是一个数组）范围从 -128 到 127，+1 是因为有个 0
    	static final Long cache[] = new Long[-(-128) + 127 + 1];

    	// 容器初始化时，进行加载
    	static {
        	// 缓存 Long 值，注意这里是 i - 128 ，所以再拿的时候就需要 + 128
        	for(int i = 0; i < cache.length; i++)
            	cache[i] = new Long(i - 128);
    	}
	}
    
}
```
## 1.1 包装类核心一：包装
Long 就是将 long 类型的 value 进行了封装与扩展

## 1.2 包装类核心二：缓存

Long 自己实现了一种缓存机制，缓存了从 -128 到 127 内的所有 Long 值，如果是这个范围内的 Long 值，就不会初始化，而是从缓存中拿；缓存在Long被加载进虚拟机时执行

可以看到，设计类的缓存池思路：**静态容器 + 静态初始化 ==> 封装成一个静态内部类**。那这里有一个问题，为什么用静态内部类，而不是分静态内部类？或者说静态内部类与非静态内部类的区别：

1. **是否能有静态成员**
   - 静态内部类可以有静态成员（属性，方法），比如上面的 `static final Long cache[]`
   - 非静态内部类则不能有静态成员（属性，方法）
2. 访问外部类的成员
   - 静态内部类只能方位外部类的静态成员
   - 非静态内部类可以访问外部类的所有成员（属性，方法）
3. 创建方式
   假设类A有静态内部类B和非静态内部类C，创建B和C的区别为：
   - 静态内部类 ：A.B b= new A.B()
   - 非静态内部类：A.C c = a.new C()


# 2.方法解析&api

## 2.1 valueOf
```java
 public static Long valueOf(long l) {
        final int offset = 128;
     	// 先在缓冲池中取
        if (l >= -128 && l <= 127) {
            // 这里+offest是要计算出索引，因为cache[0]=-127
            return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
}
```

## 2.2 longValue & intValue
```java
public long longValue() {
    	// 直接返回value
        return value;
}
```
intValue，另外还有floatValue，doubleValue
```java
 public int intValue() {
    		// 直接强制类型装换
            return (int)value;
    }
```

# 3.两个问题

1. 自动装箱与自动拆箱？

	每种基本类型都有自己的包装类，同样有自己的缓冲池，此拿long举例

	**long --`作为函参（装箱）`--> Long --`参加运算（拆箱）`--> long**
	* 自动装箱会在编译时自动调用valueOf方法
	* 自动拆箱会调用longValue方法

2. 为什么使用 Long 时，大家推荐多使用 valueOf 方法，少使用 parseLong 方法？
	因为 Long 本身有缓存机制，缓存了 -128 到 127 范围内的 Long，valueOf 方法会从缓存中去拿值，如果命中缓存，会减少资源的开销，parseLong 方法就没有这个机制。




