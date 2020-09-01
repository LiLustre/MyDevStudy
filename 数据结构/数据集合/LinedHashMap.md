### 概述
1、LinkedHashMap 是一个关联数组、哈希表，它是线程不安全的，允许key为null,value为null。

2、
继承自HashMap，实现了Map<K,V>接口。其内部还维护了一个双向链表，在每次插入数据，或者访问、修改数据时，会增加节点、或调整链表的节点顺序。以决定迭代时输出的顺序

3、遍历时的顺序是按照插入节点的顺序。这也是其与HashMap最大的区别

4、也可以在构造时传入accessOrder参数，使得其遍历顺序按照访问的顺序输出。

5、因继承自HashMap,所以HashMap上文分析的特点，除了输出无序，其他LinkedHashMap都有，比如扩容的策略，哈希桶长度一定是2的N次方等等。
LinkedHashMap在实现时，就是重写override了几个方法。以满足其输出序列有序的需求

### 节点

LinkedHashMap的节点Entry<K,V>继承自HashMap.Node<K,V>，在其基础上扩展了一下。改成了一个双向链表。

```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

同时类里有两个成员变量head tail,分别指向内部双向链表的表头、表尾。
```java
    //双向链表的头结点
    transient LinkedHashMap.Entry<K,V> head;

    //双向链表的尾节点
    transient LinkedHashMap.Entry<K,V> tail;
```
### 构造函数
    构造函数和HashMap相比，就是增加了一个accessOrder参数。用于控制迭代时的节点顺序。
```
    //默认是false，则迭代时输出的顺序是插入节点的顺序。若为true，则输出的顺序是按照访问节点的顺序。
    //为true时，可以在这基础之上构建一个LruCach
    final boolean accessOrder;
    
    public LinkedHashMap() {
        super();
        accessOrder = false;
    }
    //指定初始化时的容量，
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }
    //指定初始化时的容量，和扩容的加载因子
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }
    //指定初始化时的容量，和扩容的加载因子，以及迭代输出节点的顺序
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    //利用另一个Map 来构建，
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        //该方法上文分析过，批量插入一个map中的所有数据到 本集合中。
        putMapEntries(m, false);
    }
    
```

### 增加

1、LinkedHashMap并没有重写任何put方法。但是其重写了构建新节点的newNode()方法.
newNode()会在HashMap的putVal()方法里被调用，putVal()方法会在批量插入数据putMapEntries(Map<? extends K, ? extends V> m, boolean evict)或者插入单个数据public V put(K key, V value)时被调用。

LinkedHashMap重写了newNode(),在每次构建新节点时，通过linkNodeLast(p);将新节点链接在内部双向链表的尾部。

```java
    //在构建新节点时，构建的是`LinkedHashMap.Entry` 不再是`Node`.
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    //将新增的节点，连接在链表的尾部
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        //集合之前是空的
        if (last == null)
            head = p;
        else {//将新节点连接在链表的尾部
            p.before = last;
            last.after = p;
        }
    }
```
HashMap专门预留给LinkedHashMap的afterNodeAccess() afterNodeInsertion() afterNodeRemoval() 方法。

```java

HashMap

    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }

```
```java
LinkedHashMap
    //回调函数，新节点插入之后回调 ， 根据evict 和   判断是否需要删除最老插入的节点。如果实现LruCache会用到这个方法。
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        //LinkedHashMap 默认返回false 则不删除节点
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
    //LinkedHashMap 默认返回false 则不删除节点。 返回true 代表要删除最早的节点。通常构建一个LruCache会在达到Cache的上限是返回true
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }

```

### 删

1、LinkedHashMap也没有重写remove()方法，因为它的删除逻辑和HashMap并无区别。

2、但它重写了afterNodeRemoval()这个回调方法。该方法会在Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)方法中回调，

3、removeNode()会在所有涉及到删除节点的方法中被调用，上文分析过，是删除节点操作的真正执行者

```java
    //在删除节点e时，同步将e从双向链表上删除
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //待删除节点 p 的前置后置节点都置空
        p.before = p.after = null;
        //如果前置节点是null，则现在的头结点应该是后置节点a
        if (b == null)
            head = a;
        else//否则将前置节点b的后置节点指向a
            b.after = a;
        //同理如果后置节点时null ，则尾节点应是b
        if (a == null)
            tail = b;
        else//否则更新后置节点a的前置节点为b
            a.before = b;
    }
```

### 查
1、LinkedHashMap重写了get()和getOrDefault()方法：

2、对比HashMap中的实现,LinkedHashMap只是增加了在成员变量(构造函数时赋值)accessOrder为true的情况下，要去回调void afterNodeAccess(Node<K,V> e)函数

3、在afterNodeAccess()函数中，会将当前被访问到的节点e，移动至内部的双向链表的尾部。
```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }
```

### containsValue

1、相比HashMap的实现，更为高效。
2、HashMap，是用两个for循环遍历，相对低效。
```java
    public boolean containsValue(Object value) {
        //遍历一遍链表，去比较有没有value相等的节点，并返回
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```
HashMap
```java
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

### 总结
1、accessOrder ,默认是false，则迭代时输出的顺序是插入节点的顺序。若为true，则输出的顺序是按照访问节点的顺序。为true时，可以在这基础之上构建一个LruCache.

2、LinkedHashMap并没有重写任何put方法。但是其重写了构建新节点的newNode()方法.在每次构建新节点时，将新节点链接在内部双向链表的尾部

3、accessOrder=true的模式下,在afterNodeAccess()函数中，会将当前被访问到的节点e，移动至内部的双向链表的尾部。值得注意的是，afterNodeAccess()函数中，会修改modCount,因此当你正在accessOrder=true的模式下,迭代LinkedHashMap时，如果同时查询访问数据，也会导致fail-fast，因为迭代的顺序已经改变。

4、nextNode() 就是迭代器里的next()方法 。
该方法的实现可以看出，迭代LinkedHashMap，就是从内部维护的双链表的表头开始循环输出。

5、它与HashMap比，还有一个小小的优化，重写了containsValue()方法，直接遍历内部链表去比对value值是否相等。