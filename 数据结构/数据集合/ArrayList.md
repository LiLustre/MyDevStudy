[原文](https://www.jianshu.com/p/192148ddab9f)

### 概述

ArrayList是一个动态数组，它的底层数据结构是数组，他实现了 `List<E>、RandomAccess、 Cloneable、java.io.Serializable`接口，其中`RandomAccess`代表了其拥有**随机快速访问**的能力，`ArrayList'可以以O(1)的时间复杂度去根据下标访问元素。

- 底层数据结构：底层数据结构是数组

- ArrayList容量：

  因为它底层数据结构是数组，所以可想而知，它是有一个容量的（数组的`length`），当集合中的元素超出这个容量，便会进行扩容操作。

- ArrayList扩容：

1. 已知元素个数，指定初始容量，减少扩容次数：

   ​			扩容操作也是`ArrayList` 的一个性能消耗比较大的地方，所以若**我们可以提前预知数据的规模**，应该通过`public ArrayList(int initialCapacity) {}`构造方法，			指定集合的大小，去构建`ArrayList`实例**，以减少扩容次数，提高效率**。

2. 调研Api手动扩容：

   ​			在需要扩容的时候，手动调用`public void ensureCapacity(int minCapacity) {}`方法扩容。不过该方法是`ArrayList`的API，不是`List`接口里的，所以使用时			需要强转:`((ArrayList)list).ensureCapacity(30);`

### 源码分析

#### 一、构造函数

1.构造函数执行完会根据初始容量创建出真正存储数据的elementData 数组，并初始化size 大小，

2.如果使用集合为参数的构造函数则用`Collection.toArray()` 方法先将集合转换数组并赋值给elementData ，size 大小 为集合内元素数量的大小

源码如下

```java
    //默认构造函数里的空数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    //存储集合元素的底层实现：真正存放元素的数组
    transient Object[] elementData; // non-private to simplify nested class access
    //当前元素数量
    private int size;
    
    //默认构造方法
    public ArrayList() {
        //默认构造方法只是简单的将 空数组赋值给了elementData
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    //空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    //带初始容量的构造方法
    public ArrayList(int initialCapacity) {
        //如果初始容量大于0，则新建一个长度为initialCapacity的Object数组.
        //注意这里并没有修改size(对比第三个构造函数)
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//如果容量为0，直接将EMPTY_ELEMENTDATA赋值给elementData
            this.elementData = EMPTY_ELEMENTDATA;
        } else {//容量小于0，直接抛出异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    //利用别的集合类来构建ArrayList的构造函数
    public ArrayList(Collection<? extends E> c) {
        //直接利用Collection.toArray()方法得到一个对象数组，并赋值给elementData 
        elementData = c.toArray();
        //因为size代表的是集合元素数量，所以通过别的集合来构造ArrayList时，要给size赋值
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)//这里是当c.toArray出错，没有返回Object[]时，利用Arrays.copyOf 来复制集合c中的元素到elementData数组中
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            //如果集合c元素数量为0，则将空数组EMPTY_ELEMENTDATA赋值给elementData 
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

**Collection.toArray()**：这个方法，在Collection子类各大集合的源码中，高频使用了这个方法去获得某Collection的所有元素

**Arrays.copyOf(elementData, size, Object[].class)**：就是根据class的类型来决定是new 还是反射去构造一个泛型数组，同时利用native函数，批量赋值元素至新数组中

```java
    public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        //根据class的类型来决定是new 还是反射去构造一个泛型数组
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        //利用native函数，批量赋值元素至新数组中。
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

**System.arraycopy()**：也是一个很高频的函数 

```java
/*
	 * @param      src     数据源，要cope的数据源
     * @param      srcPos  数据源要cope的数组的起始索引
     * @param      dest     要copy 到的目的数据结构
     * @param      destPos 目的数据结构的存储copy数据的开始索引
     * @param      length   要cope的数据源的长度
*/
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

