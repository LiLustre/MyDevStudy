[原文](https://www.jianshu.com/p/22ae6596b004)

### 概述

`1、HashMap` 是一个**关联数组、哈希表**，它是**线程不安全**的，允许**key为null**,**value为null**。遍历时**无序**，它实现了`Map<K,V>, Cloneable, Serializable`接口。

2、其底层数据结构是**数组**称之为**哈希桶**，每个**桶里面放的是链表**，链表中的**每个节点**，就是哈希表中的**每个元素**。因其底层哈希桶的数据结构是数组，所以也会涉及到**扩容**的问题。

3、在JDK8中，当链表长度达到8，会转化成红黑树，以提升它的查询、插入效率

4、当`HashMap`的容量达到`threshold`域值时，就会触发扩容。扩容前后，哈希桶的**长度一定会是2的次方**

**key的hash值获取数组索引：**

1、key的hash值，并不仅仅只是key对象的`hashCode()`方法的返回值，还会经过**扰动函数**的扰动，以使hash值更加均衡。扰动函数 在HashMap  的`  hash(Object key)`

2、`hashCode()`是`int`类型，取值范围是40多亿，但是由于`HashMap`的哈希桶的长度远比hash取值范围小，默认是16，所以当对hash值以桶的长度取余，
3、**扰动函数**就是为了解决hash碰撞的。它会综合hash值高位和低位的特征，并存放在低位，因此在与运算时，相当于高低位一起参与了运算，以减少hash碰撞的概率

数据结构如下
![image](图片\HashMap数据结构.png)

### 源码分析
#### 构造函数
    1、最大容量 2的30次方
    2、默认的加载因子0.75
    3、加载因子，用于计算哈希表元素数量的阈值。  threshold = 哈希桶.length * loadFactor
    4、哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()。
``` java
    //最大容量 2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认的加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    //哈希桶，存放链表。 长度是2的N次方，或者初始化时为0.
    transient Node<K,V>[] table;
        
    //加载因子，用于计算哈希表元素数量的阈值。  threshold = 哈希桶.length * loadFactor;
    final float loadFactor;
    //哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()。
    int threshold;

    public HashMap() {
        //默认构造函数，赋值加载因子为默认的0.75f
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    public HashMap(int initialCapacity) {
        //指定初始化容量的构造函数
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    //同时指定初始化容量 以及 加载因子， 用的很少，一般不会修改loadFactor
    public HashMap(int initialCapacity, float loadFactor) {
        //边界处理
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //初始容量最大不能超过2的30次方
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //显然加载因子不能为负数
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        //设置阈值为  》=初始化容量的 2的n次方的值
        this.threshold = tableSizeFor(initialCapacity);
    }
    //新建一个哈希表，同时将另一个map m 里的所有元素加入表中
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    
    //根据期望容量cap，返回2的n次方形式的 哈希桶的实际容量 length。 返回值一般会>=cap 
    static final int tableSizeFor(int cap) {
    //经过下面的 或 和位移 运算， n最终各位都是1。
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        //判断n是否越界，返回 2的n次方作为 table（哈希桶）的阈值
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
        //将另一个Map的所有元素加入表中，参数evict初始化时为false，其他情况为true
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //拿到m的元素数量
        int s = m.size();
        //如果数量大于0
        if (s > 0) {
            //如果当前表是空的
            if (table == null) { // pre-size
                //根据m的元素数量和当前表的加载因子，计算出阈值
                float ft = ((float)s / loadFactor) + 1.0F;
                //修正阈值的边界 不能超过MAXIMUM_CAPACITY
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                //如果新的阈值大于当前阈值
                if (t > threshold)
                    //返回一个 》=新的阈值的 满足2的n次方的阈值
                    threshold = tableSizeFor(t);
            }
            //如果当前元素表不是空的，但是 m的元素数量大于阈值，说明一定要扩容。
            else if (s > threshold)
                resize();
            //遍历 m 依次将元素加入当前表中。
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

### 与HashTable的区别
- 与之相比HashTable是线程安全的，且不允许key、value是null。
- HashTable默认容量是11。
- HashTable是直接使用key的hashCode(key.hashCode())作为hash值，不像HashMap内部使用static final int hash(Object key)扰动函数对key的hashCode进行扰动后作为hash值。
- HashTable取哈希桶下标是直接用模运算%.（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算）
- 扩容时，新容量是原来的2倍+1。int newCapacity = (oldCapacity << 1) + 1;
- Hashtable是Dictionary的子类同时也实现了Map接口，HashMap是Map接口的一个实现类；

