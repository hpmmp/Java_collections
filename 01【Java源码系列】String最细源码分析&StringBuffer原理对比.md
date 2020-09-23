# 1.结构
String 继承关系，核心成员变量，主要构造函数：
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /**
     * 1.final会被jvm缓存，提高了性能
     * 2.fianl变量线程安全，节省了线程同步的开销
     * 正因为是final的，所有不可变，即所有String都是新的
     * 注：这个数组是不可变的，不存在容器的扩容问题
     */
    private final char value[];
    
    // 空参构造 如果只是new String()那么只会产生一个空数组
    public String() {
      this.value = new char[0];
	}
    
    // char[]构造，调用 Arrays.copyOf
    // 注：Arrays.copyOf是拷贝后返回一个新数组，system.arrayCopy是可以对目标数组拷贝
    public String(char value[]) {
    	this.value = Arrays.copyOf(value, value.length);
	}
   
    // 常用于subString方法，实际是调用Arrays.copyOfRange(部分拷贝)
    public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
    
    // 构造器有很多，这里只列举3个.....
}    
```

## 1.1 实现的接口

- Serializable : 序列化，string可以写到io流中，并可保存整个对象以及用于网络传输
```java
private static final long serialVersionUID = -6849794470754667710L;
```
- Comparable : 比较，若返回值>0则，当前字符串小于另一个字符串
```java
// 注：只有在实现Compareable时指定了泛型，这里才会是myString ====> 指定泛型是必要的
public int compareTo(String anotherString) {
        int len1 = value.length;
        int len2 = anotherString.value.length;
        int lim = Math.min(len1, len2);
        char v1[] = value;
        char v2[] = anotherString.value;

    	// 1.先逐个字符比较，若遇见不同了，即使A比B长，但B[i]>A[i]，那么也是B>A
        int k = 0;
        while (k < lim) {
            char c1 = v1[k];
            char c2 = v2[k];
            // 不相同，就ASCII相减
            if (c1 != c2) {
                return c1 - c2;
            }
            k++;
        }
    	// 2.再比较长度
        return len1 - len2;
    }
```
- CharSequence：CharSequence与String都能用于定义字符串，但CharSequence的值是可读可写序列，而String的值是只读序列

## 1.2  final 与不变性

- String 被 final 修饰，说明 String 类绝不可能被继承了，也就是说任何对 String 的操作方法，都不会被继承覆写
- String 中保存数据的是一个 char 的数组 value。我们发现 value 也是被 final 修饰的，也就是说 value 一旦被赋值，内存地址是绝对无法修改的，而且 value 的权限是 private 的，外部绝对访问不到，String 也没有开放出可以对 value 进行赋值的方法，所以说 value 一旦产生，内存地址就根本无法被修改。
- final的作用 
  - final关键字提高了性能，JVM缓存了final变量
  - final变量线程安全，节省了多线程环境下同步的开销
  - JVM会对final方法及变量进行优化

String通过充分利用final关键字实现了不变性，所以string的绝大多数写法都是返回新的string

两个string不变性的示例：

- demo1:
```java
String str ="hello world !!";
// 这种写法是替换不掉的，必须接受 replace 方法返回的参数才行，这样才行：str = str.replace("l","dd");
str.replace("l","dd");
```
- demo2:
```java
String s ="hello";
s ="world";
```
从代码上来看，s 的值好像被修改了，但从 debug 的日志来看，其实是 s 的内存地址已经被修改了，也就说 s =“world” 这个看似简单的赋值，其实已经把 s 的引用指向了新的 String，debug 的截图显示内存地址已经被修改，两张截图如下：
<img src="https://img1.sycdn.imooc.com/5d5fc04a0001c6a508840096.png" width="70%">
<img src="https://img1.sycdn.imooc.com/5d5fc06400019cc210540090.png" width="70%">
## 1.3 new与不new的问题
```java
String str = "abc";
String str1 = new String("abc");
System.out.println(str == str1); // false
System.out.println(str.equals(str1)); // true
```
- str是直接将一个String对象附给str，由于底层是final，所以会将常量池（方法区）的已有对象返回，若不存在则会现在常量池中创建然后将地址返回
- str1使用了new，将会在堆内存中新开辟一块内存用来存储

# 2.方法解析&api

## 2.1 equals（比较）
```java
public boolean equals(Object anObject) {
        // 1.判断内存地址是否相同
        if (this == anObject) {
            return true;
        }
        // 2.待比较的对象是否是 String，如果不是 String，直接返回不相等
        if (anObject instanceof String) {
            // 注：instance使其父类也为true
            String anotherString = (String)anObject;
            int n = value.length;
            // 3.两个字符串的长度是否相等，不等直接返回不相等
            // 注：同一个类中可以直接调用private属性
            if (n == anotherString.value.length) {
                // 注：这里先用一个数组存下来value是为了下面代码的可读性
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                // 4.依次比较每个字符是否相等，若有一个不等，直接返回不相等
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```
## 2.2 subString（截取）
```java
public String substring(int beginIndex, int endIndex) {  // 左闭右开
	if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        if (endIndex > value.length) {
            throw new StringIndexOutOfBoundsException(endIndex);
        }
        int subLen = endIndex - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
    	// 调用构造函数
        return ((beginIndex == 0) && (endIndex == value.length)) ? this
                : new String(value, beginIndex, subLen);
}
```
```java
public String substring(int beginIndex)
```
- substring 方法的底层使用的是字符数组范围截取的方法 ：Arrays.copyOfRange(字符数组, 开始位置, 结束位置); 从字符数组中进行一段范围的拷贝。
- 返回一个新的String

## 2.3 replace & replaceAll（替换）
```java
public String replace(char oldChar, char newChar) { // 字符替换，全部
        if (oldChar != newChar) {
            int len = value.length;
            int i = -1;
            char[] val = value; /* avoid getfield opcode */
			
            // 1.整体遍历一遍，看有无需要被替换字符
            while (++i < len) {
                if (val[i] == oldChar) {
                    break;
                }
            }
            // 2.1 若有，再创建一个新数组，然后替换后放入
            if (i < len) {
                char buf[] = new char[len];
                // 注：这里是先将 i 之前的直接赋给新数组buf
                for (int j = 0; j < i; j++) {
                    buf[j] = val[j];
                }
                while (i < len) {
                    char c = val[i];
                    // 注：这里一个？:就搞定
                    buf[i] = (c == oldChar) ? newChar : c;
                    i++;
                }
                // 重新构造一个String
                return new String(buf, true);
            }
        }
    	// 2.2 若无，则直接返回本身
        return this;
    }
```
```java
public String replaceFirst(String regex, String replacement) { // 只替换第一次出现的字符串
      // 正则表达式
      return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}
```
```java
public String replaceAll(String regex, String replacement) { // 字符串替换，全部
        return Pattern.compile(regex).matcher(this).replaceAll(replacement);
    }
```
- 当然我们想要删除某些字符，也可以使用 replace 方法，把想删除的字符替换成 “” 即可。
- 首字母大写转小写： name.substring(0, 1).toUpperCase()/toLowerCase() + name.substring(1)

## 2.4 split & join
### split （分割）
字符串分割，返回string[]
```java
public String[] split(String regex, int limit) // regex:分隔符 ，limit：拆分的个数
public String[] split(String regex) // return split(regex, 0);
```
示例：对字符串 s 进行各种拆分
 ```java
 String s ="boo:and:foo";
// 演示的代码和结果是：
s.split(":")  // 结果:["boo","and","foo"]
s.split(":",2) // 结果:["boo","and:foo"]
s.split(":",5) // 结果:["boo","and","foo"]
s.split(":",-2) // 结果:["boo","and","foo"]
s.split("o") // 结果:["b","",":and:f"]
s.split("o",2) // 结果:["b","o:and:foo"]
```
  
### join （连接）

字符串拼接，返回字符串
```java
// delimiter:分隔符，elements:数据（list/array)
public static String join(CharSequence delimiter, CharSequence... elements) {
        Objects.requireNonNull(delimiter);
        Objects.requireNonNull(elements);
        // 底层是StringBuilder.append（）
        StringJoiner joiner = new StringJoiner(delimiter);
        for (CharSequence cs: elements) {
            joiner.add(cs);
        }
        return joiner.toString();
    }
```
示例：
```java
List<String> names=new ArrayList<String>();
names.add("1");
names.add("2");
names.add("3");
System.out.println(String.join("-", names)); // 1-2-3
 
String[] arrStr=new String[]{"a","b","c"};
System.out.println(String.join("-", arrStr)); // a-b-c
```
## 2.5 indexOf & charAt （查找）
### indexOf（返回索引）
indexOf返回index，没找到就返回-1
```java
public int indexOf(int ch, int fromIndex) {
    final int max = value.length;
    if (fromIndex < 0) {
        fromIndex = 0;
    } else if (fromIndex >= max) {
        // Note: fromIndex might be near -1>>>1.
        return -1;
    }

    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
        final char[] value = this.value;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return indexOfSupplementary(ch, fromIndex);
    }
}
```
### charAt（返回字符）
charAt返回指定索引的char
```java
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```
## 2.6 toCharArray  & getBytes
### toCharArray（->char数组）
底层就是char数组存储的，所以直接返回char数组就行

注：这里不能直接将value返回，因为value是final不可变的，那么返回后使用者也无法操作
```java
public char[] toCharArray() {
       	// 新建一个数组，拷贝过去返回
        char result[] = new char[value.length];
    	// arraycopy(src, srcPos, dest, destPos, length)
        System.arraycopy(value, 0, result, 0, value.length);
        return result;
}
```
### getBytes（->二进制数组）

在生活中，我们经常碰到这样的场景，进行二进制转化操作时，本地测试的都没有问题，到其它环境机器上时，有时会出现字符串乱码的情况，这个主要是因为在二进制转化操作时，并没有强制规定文件编码，而不同的环境默认的文件编码不一致导致的。

我们也写了一个 demo 来模仿一下字符串乱码：
```java
String str  ="nihao 你好 喬亂";
// 字符串转化成 byte 数组，转化成ISO-8859-1编码的二进制数组
// 注：ISO-8859-1编码格式不能支持所有的汉字，所以大概率乱码
byte[] bytes = str.getBytes("ISO-8859-1");
// byte 数组转化成字符串
String s2 = new String(bytes);
log.info(s2); 
```
```
 结果打印为：nihao ?? ??
```
打印的结果为？？，这就是常见的乱码表现形式。是不是我把代码修改成 String s2 = new String(bytes,"ISO-8859-1"); 就可以了？这是不行的。主要是因为 ISO-8859-1 这种编码对中文的支持有限，导致中文会显示乱码。唯一的解决办法，就是在所有需要用到编码的地方，都统一使用 UTF-8，对于 String 来说，getBytes 和 new String 两个方法都会使用到编码，我们把这两处的编码替换成 UTF-8 后，打印出的结果就正常了。

## 2.7  valueOf & toString
转换String：new String(~~)   或   valueOf()    或    toString() 

```java
// 注：value是个静态方法，可以将基本类型和Object转换为String
public static String valueOf(Object obj) {
    // 转换对象时调用的也是toString，但允许null
    // 因此在做一些操作时要慎重，如Integer.valueOf(str);这是就会报错提示类型转换出错
    return (obj == null) ? "null" : obj.toString(); // 当obj==null时，"null"
}
```
```java
public static String valueOf(int i) {
    // 先装箱成Integer再转换为String
    return Integer.toString(i);
}
```
**总结：对象 --> String ：toString      基本类型 --> new String（~~）**

## 2.8 getChars（拷贝）

将当前字符串（value）拷贝到另一个数组（dst）
```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    // 直接调用System.arraycopy，value就是当前字符串
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```
# 3.两个问题

**StringBuilder原理？为什么拼接效率高于String？**

StringBuilder的方法实质上是调用父类AbstractStringBuilder的

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
   
    // 实质上也是char[]，但他不是final的==>也就是说，在数组空间够大的情况下，一个数组可以存多个字符串
    char[] value;

    // 记录当前容量，扩容的前提
    int count;
}
```
StringBuilder的append方法实际上就是调用父类AbstractStringBuilder的append方法

   ```java
   public StringBuilder(String str) {
    super(str.length() + 16);
    append(str);
}
   ```
```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    // 确保容量够，不够则扩容
    // 扩容机制是2*capcity+2
    ensureCapacityInternal(count + len);
    // 将要append的string拷贝到当前数组，0表示从要拼接的字符串第一位开始拷贝
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
   ```

接效率高的原因？

1. String是final的，一个数组只能给一个字符串使用，所以每拼接一次都要新创建然后拷贝一次数组
2. StringBuilder虽然底层也是字符数组，但是他不final，即允许在容量充足的情况下，一个数组可以被拼接多次；
   相应的StringBuilder也要有扩容机制

**StringBuilder与StringBuffer的区别**

StringBuffer线程安全，在append方法前面加上了synchronized，但相对效率就低了
```java
public synchronized StringBuffer append(StringBuffer sb) {
    toStringCache = null;
    super.append(sb);
    return this;
}
```


