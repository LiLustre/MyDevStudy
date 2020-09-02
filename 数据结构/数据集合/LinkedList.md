### 概述
1、ArrayList 和 LinkedList 可以说是List接口的两种不同的实现，ArrayList的增删效率低，但是改查效率高

2、LinkedList正好相反，增删由于不需要移动底层数组数据，其底层是链表实现的，只需要修改链表节点指针，所以效率较高。

3、而改和查，都需要先for循环定位到目标节点 所以效率较低。

4、LinkedList 是线程不安全的，允许元素为null的双向链表

5、底层数据结构是链表，它实现List<E>, Deque<E>, Cloneable, java.io.Serializable接口，它实现了Deque<E>,所以它也可以作为一个双端队列。和ArrayList比，没有实现RandomAccess所以其以下标，随机访问元素速度较慢。

优点：

1、增删快 因其底层数据结构是链表，所以可想而知，它的增删只需要移动指针即可，故时间效率较高。

2、不需要批量扩容，也不需要预留空间，所以空间效率比ArrayList高

缺点：

需要随机访问元素时，时间效率很低，虽然底层在根据下标查询Node的时候，会根据index判断目标Node在前半段还是后半段，然后决定是顺序还是逆序查询，以提升时间效率。不过随着n的增大，总体时间效率依然很低

LinkedList的数据结构如图
![image](https://github.com/LiLustre/MyDevStudy/blob/master/%E5%9B%BE%E7%89%87/linkedList%E9%93%BE%E8%A1%A8%E7%BB%93%E6%9E%84.png?raw=true)


### 源码分析 
#### 构造函数、节点

```java
    //集合元素数量
    transient int size = 0;
    //链表头节点
    transient Node<E> first;
    //链表尾节点
    transient Node<E> last;
    //啥都不干
    public LinkedList() {
    }
    //调用     public boolean addAll(Collection<? extends E> c)  将集合c所有元素插入链表中
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

private static class Node<E> {
        E item;//元素值
        Node<E> next;//后置节点
        Node<E> prev;//前置节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
 }
```

#### 增加

基本结构如图所示
![image](https://github.com/LiLustre/MyDevStudy/blob/master/%E5%9B%BE%E7%89%87/LinkedList%E6%B7%BB%E5%8A%A0%E5%85%83%E7%B4%A0.png?raw=true)

##### 批量增加
1、链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。对比ArrayList是通过System.arraycopy完成批量增加的

2、通过下标获取某个node 的时候，（add select），会根据index处于前半段还是后半段 进行一个折半，以提升查询效率

```java
//addAll ,在尾部批量增加
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);//以size为插入下标，插入集合c中所有元素
}
//以index为插入下标，插入集合c中所有元素
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);//检查越界 [0,size] 闭区间

    Object[] a = c.toArray();//拿到目标集合数组
    int numNew = a.length;//新增元素的数量
    if (numNew == 0)//如果新增元素数量为0，则不增加，并返回false
        return false;

    Node<E> pred, succ;  //index节点的前置节点，后置节点
    if (index == size) { //在链表尾部追加数据
        succ = null;  //size节点（队尾）的后置节点一定是null
        pred = last;//前置节点是队尾
    } else {
        succ = node(index);//取出index节点，作为后置节点
        pred = succ.prev; //前置节点是，index节点的前一个节点
    }
    //链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。对比ArrayList是通过System.arraycopy完成批量增加的
    for (Object o : a) {//遍历要添加的节点。
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);//以前置节点 和 元素值e，构建new一个新节点，
        if (pred == null) //如果前置节点是空，说明是头结点
            first = newNode;
        else//否则 前置节点的后置节点设置问新节点
            pred.next = newNode;
        pred = newNode;//步进，当前的节点为前置节点了，为下次添加节点做准备
    }

    if (succ == null) {//循环结束后，判断，如果后置节点是null。 说明此时是在队尾append的。
        last = pred; //则设置尾节点
    } else {
        pred.next = succ; // 否则是在队中插入的节点 ，更新前置节点 后置节点
        succ.prev = pred; //更新后置节点的前置节点
    }

    size += numNew;  // 修改数量size
    modCount++;  //修改modCount
    return true;
}
//根据index 查询出Node，
Node<E> node(int index) {
    // assert isElementIndex(index);
//通过下标获取某个node 的时候，（增、查 ），会根据index处于前半段还是后半段 进行一个折半，以提升查询效率
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;  //插入时的检查，下标可以是size [0,size]
}
```
##### 插入单个节点
1、生成新节点 并插入到 链表尾部， 更新 last/first 节点。

2、按照索引位置插入，找出该位置的节点，修改该节点的pre 和该位置前一个位置的节点的next，并设置该位置节点的pre 和next

``` java
//在尾部插入一个节点： add
public boolean add(E e) {
    linkLast(e);
    return true;
}

//生成新节点 并插入到 链表尾部， 更新 last/first 节点。
void linkLast(E e) { 
    final Node<E> l = last; //记录原尾部节点
    final Node<E> newNode = new Node<>(l, e, null);//以原尾部节点为新节点的前置节点
    last = newNode;//更新尾部节点
    if (l == null)//若原链表为空链表，需要额外更新头结点
        first = newNode;
    else//否则更新原尾节点的后置节点为现在的尾节点（新节点）
        l.next = newNode;
    size++;//修改size
    modCount++;//修改modCount
}
```
```java
    //在指定下标，index处，插入一个节点
    public void add(int index, E element) {
        checkPositionIndex(index);//检查下标是否越界[0,size]
        if (index == size)//在尾节点后插入
            linkLast(element);
        else//在中间插入
            linkBefore(element, node(index));
    }
    //在succ节点前，插入一个新节点e
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        //保存后置节点的前置节点
        final Node<E> pred = succ.prev;
        //以前置和后置节点和元素值e 构建一个新节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        //新节点new是原节点succ的前置节点
        succ.prev = newNode;
        if (pred == null)//如果之前的前置节点是空，说明succ是原头结点。所以新节点是现在的头结点
            first = newNode;
        else//否则构建前置节点的后置节点为new
            pred.next = newNode;
        size++;//修改数量
        modCount++;//修改modCount
    }
```

#### 删除
1、按照索引删除
    
    1、检查是否越界 下标[0,size)
    2、从链表上删除某节点 （通过修改当前节点的前置节点的next 和 当前节点后置节点pre 来说实现删除，最后将当前节点置空）

```java
//删：remove目标节点
public E remove(int index) {
    checkElementIndex(index);//检查是否越界 下标[0,size)
    return unlink(node(index));//从链表上删除某节点
}
//从链表上删除x节点
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item; //当前节点的元素值
    final Node<E> next = x.next; //当前节点的后置节点
    final Node<E> prev = x.prev;//当前节点的前置节点

    if (prev == null) { //如果前置节点为空(说明当前节点原本是头结点)
        first = next;  //则头结点等于后置节点 
    } else { 
        prev.next = next;
        x.prev = null; //将当前节点的 前置节点置空
    }

    if (next == null) {//如果后置节点为空（说明当前节点原本是尾节点）
        last = prev; //则 尾节点为前置节点
    } else {
        next.prev = prev;
        x.next = null;//将当前节点的 后置节点置空
    }

    x.item = null; //将当前元素值置空
    size--; //修改数量
    modCount++;  //修改modCount
    return element; //返回取出的元素值
}
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    //下标[0,size)
    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }

```

2、通过元素删除
    
    1、获取当前元素的node
    2、同上执行删除操作
```java
    //因为要考虑 null元素，也是要遍历两次
    public boolean remove(Object o) {
        if (o == null) {//如果要删除的是null节点(从remove和add 里 可以看出，允许元素为null)
            //遍历每个节点 对比
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
    //将节点x，从链表中删除
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;//继续元素值，供返回
        final Node<E> next = x.next;//保存当前节点的后置节点
        final Node<E> prev = x.prev;//前置节点

        if (prev == null) {//前置节点为null，
            first = next;//则首节点为next
        } else {//否则 更新前置节点的后置节点
            prev.next = next;
            x.prev = null;//记得将要删除节点的前置节点置null
        }
        //如果后置节点为null，说明是尾节点
        if (next == null) {
            last = prev;
        } else {//否则更新 后置节点的前置节点
            next.prev = prev;
            x.next = null;//记得删除节点的后置节点为null
        }
        //将删除节点的元素值置null，以便GC
        x.item = null;
        size--;//修改size
        modCount++;//修改modCount
        return element;//返回删除的元素值
    }

```

### 总结

1、链表批量增加，是靠for循环遍历原数组，依次执行插入节点操作。对比ArrayList是通过System.arraycopy完成批量增加的。增加一定会修改modCount。

2、通过下标获取某个node 的时候，（add select），会根据index处于前半段还是后半段 进行一个折半，以提升查询效率

3、按下标删，也是先根据index找到Node，然后去链表上unlink掉这个Node。 按元素删，会先去遍历链表寻找是否有该Node，然后再去unlink它。