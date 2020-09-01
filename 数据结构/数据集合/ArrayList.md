原文](https://www.jianshu.com/p/192148ddab9f)



[TOC]

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

#### 二、增加

​	增加元素分为  `add(E e)`、 `add(int index, E element)` 、`addAll(Collection<? extends E> c)`

​	每次增加元素都需要判断是否需要扩容，默认会扩容当前容量的一半，如果不够的话就就用目标的size作为扩容后的容量

​	**add(E e)**

1. 每次add之前都要判断add后是否需要扩容，需要扩容的话，默认扩容当前容量的一半，扩容既创建新数组，并将原数据copy到新数组中

2. 将元素追加到size++ 的索引位置

   ```
   public boolean add(E e) {
       ensureCapacityInternal(size + 1);  // Increments modCount!!
       elementData[size++] = e;//在数组末尾追加一个元素，并修改size
       return true;
   }
       private static final int DEFAULT_CAPACITY = 10;//默认扩容容量 10
       private void ensureCapacityInternal(int minCapacity) {
           //利用 == 可以判断数组是否是用默认构造函数初始化的
           if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
               minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
           }
   
           ensureExplicitCapacity(minCapacity);
       }
   
   
   private void ensureExplicitCapacity(int minCapacity) {
       modCount++;//如果确定要扩容，会修改modCount 
   
       // overflow-conscious code
       if (minCapacity - elementData.length > 0)
           grow(minCapacity);
   }
   
   //需要扩容的话，默认扩容一半
   private void grow(int minCapacity) {
       // overflow-conscious code
       int oldCapacity = elementData.length;
       int newCapacity = oldCapacity + (oldCapacity >> 1);//默认扩容一半
       if (newCapacity - minCapacity < 0)//如果还不够 ，那么就用 能容纳的最小的数量。（add后的容量）
           newCapacity = minCapacity;
       if (newCapacity - MAX_ARRAY_SIZE > 0)
           newCapacity = hugeCapacity(minCapacity);
       // minCapacity is usually close to size, so this is a win:
       elementData = Arrays.copyOf(elementData, newCapacity);//拷贝，扩容，构建一个新数组，
   }
   ```

   ​	**add(int index, E element)**

   1. 越界判断 如果越界抛异常

   2. 判断add后是否需要扩容，需要扩容的话，默认扩容当前容量的一半，扩容既创建新数组，并将原数据copy到新数组中

   3. 将index开始的数据 向后移动一位

   4. 将元素增加到index位置的数组

      ```java
      
      public void add(int index, E element) {
          rangeCheckForAdd(index);//越界判断 如果越界抛异常
      
          ensureCapacityInternal(size + 1);  // Increments modCount!!
          System.arraycopy(elementData, index, elementData, index + 1,
                           size - index); //将index开始的数据 向后移动一位
          elementData[index] = element;
          size++;
      }
      ```

      

      **addAll(Collection<? extends E> c)**

      基本流程同上

      ```java
      public boolean addAll(int index, Collection<? extends E> c) {
          rangeCheckForAdd(index);//越界判断
      
          Object[] a = c.toArray();
          int numNew = a.length;
          ensureCapacityInternal(size + numNew);  // Increments modCount //确认是否需要扩容
      
          int numMoved = size - index;
          if (numMoved > 0)
              System.arraycopy(elementData, index, elementData, index + numNew,
                               numMoved);//移动（复制)数组
      
          System.arraycopy(a, 0, elementData, index, numNew);//复制数组完成批量赋值
          size += numNew;
          return numNew != 0;
      }
      ```

      #### 三、删除

      删除方法 分为 ` remove(int index) `、`remove(Object o)`、` removeAll(Collection<?> c) `

      批量删除时 从这里我们也可以看出，当用来作为删除元素的集合里的元素多余被删除集合时，也没事，只会删除它们共同拥有的元素。

       批量删除中，涉及高效的保存两个集合公有元素的算法，可以留意一下

      删除操作**一定会修改modCount**，且**可能涉及到数组的复制**，**相对低效**。

      

   ​		**remove(int index) **

   1. 判断是否越界

   2. 读出要删除的值

   3. 用复制 覆盖数组index位置数据

   4. 置空原尾部数据 不再强引用， 可以GC掉			

      ```
      public E remove(int index) {
          rangeCheck(index);//判断是否越界
          modCount++;//修改modeCount 因为结构改变了
          E oldValue = elementData(index);//读出要删除的值
          int numMoved = size - index - 1;
          if (numMoved > 0)
              System.arraycopy(elementData, index+1, elementData, index,
                               numMoved);//用复制 覆盖数组数据
          elementData[--size] = null; // clear to let GC do its work  //置空原尾部数据 不再强引用， 可以GC掉
          return oldValue;
      }
          //根据下标从数组取值 并强转
          E elementData(int index) {
              return (E) elementData[index];
          }
      
      //删除该元素在数组中第一次出现的位置上的数据。 如果有该元素返回true，如果false。
      public boolean remove(Object o) {
          if (o == null) {
              for (int index = 0; index < size; index++)
                  if (elementData[index] == null) {
                      fastRemove(index);//根据index删除元素
                      return true;
                  }
          } else {
              for (int index = 0; index < size; index++)
                  if (o.equals(elementData[index])) {
                      fastRemove(index);
                      return true;
                  }
          }
          return false;
      }
      //不会越界 不用判断 ，也不需要取出该元素。
      private void fastRemove(int index) {
          modCount++;//修改modCount
          int numMoved = size - index - 1;//计算要移动的元素数量
          if (numMoved > 0)
              System.arraycopy(elementData, index+1, elementData, index,
                               numMoved);//以复制覆盖元素 完成删除
          elementData[--size] = null; // clear to let GC do its work  //置空 不再强引用
      }
      
      //批量删除
      public boolean removeAll(Collection<?> c) {
          Objects.requireNonNull(c);//判空
          return batchRemove(c, false);
      }
      //批量移动
      private boolean batchRemove(Collection<?> c, boolean complement) {
          final Object[] elementData = this.elementData;
          int r = 0, w = 0;//w 代表批量删除后 数组还剩多少元素
          boolean modified = false;
          try {
              //高效的保存两个集合公有元素的算法
              for (; r < size; r++)
                  if (c.contains(elementData[r]) == complement) // 如果 c里不包含当前下标元素， 
                      elementData[w++] = elementData[r];//则保留
          } finally {
              // Preserve behavioral compatibility with AbstractCollection,
              // even if c.contains() throws.
              if (r != size) { //出现异常会导致 r !=size , 则将出现异常处后面的数据全部复制覆盖到数组里。
                  System.arraycopy(elementData, r,
                                   elementData, w,
                                   size - r);
                  w += size - r;//修改 w数量
              }
              if (w != size) {//置空数组后面的元素
                  // clear to let GC do its work
                  for (int i = w; i < size; i++)
                      elementData[i] = null;
                  modCount += size - w;//修改modCount
                  size = w;// 修改size
                  modified = true;
              }
          }
          return modified;
      }
      ```

      ### 改

      相对增删是高效的操作

      ```java
      public E set(int index, E element) {
          rangeCheck(index);//越界检查
          E oldValue = elementData(index); //取出元素 
          elementData[index] = element;//覆盖元素
          return oldValue;//返回元素
      }
      ```

      #### clear()

      将数组中的所有元素置空，将size 置0

      ​	

      ```java
      public void clear() {
          modCount++;//修改modCount
          // clear to let GC do its work
          for (int i = 0; i < size; i++)  //将所有元素置null
              elementData[i] = null;
      
          size = 0; //修改size 
      }
      ```

      

      

### 总结

1. 增删改查中 增删操作都设计到数组的复制，所以效率会低。 而 改、查都是很高效的操作。
2. 因此，结合特点，在使用中，以Android中最常用的展示列表为例，列表滑动时需要展示每一个Item（element）的数组，**所以 查 操作是最高频的**。相对来说，**增操作 只有在列表加载更多时才会用到** ，而且是在列表尾部插入，所以也不需要移动数据的操作。而删操作则更低频。 故选用ArrayList作为保存数据的结构。
3. 