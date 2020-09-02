### 概述

1、`SparseArray<E>`是用于在Android平台上替代`HashMap`的数据结构 即 用于替代`key`为`int`类型，`value`为`Object`类型的`HashMap`

2、由于key指定为`int`类型，也可以节省`int`-`Integer`的**装箱拆箱**操作带来的**性能消耗**，比`HashMap`更加**节省空间**

3、仅实现了`implements Cloneable`接口，所以使用时不能用`Map`作为声明类型来使用

4、是**线程不安全**的，允许value为null

5、**扩容时只需要数组拷贝工作，不需要重建哈希表**。

6、**不适合大容量**的数据存储。存储大量数据时，它的性能将退化至少50%

7、比传统的`HashMap`**时间效率低**。
 因为其会对key从小到大排序，使用**二分法**查询key对应在数组中的下标。
 在添加、删除、查找数据的时候都是**先使用二分查找法得到相应的index**，然后通过index来进行添加、查找、删除等操作

**注意**

`SparseArray`**为了提升性能**，在**删除操作时**做了一些**优化**：
 当删除一个元素时，并不是立即从`value`数组中删除它，并压缩数组，
 而是将其在`value`数组中**标记为已删除**。这样当存储相同的`key`的`value`时，可以**重用**这个空间。
 如果该空间没有被重用，随后将在合适的时机里执行gc（垃圾收集）操作，将数组压缩，以免浪费空间。

### 组成实现

它的内部实现也是**基于两个数组**。

1、一个`int[]`数组`mKeys`，用于保存每个item的`key`，`key`本身就是`int`类型，所以可以理解`hashCode`值就是`key`的值.

2、一个`Object[]`数组`mValues`，保存`value`。**容量**和`key`数组的**一样**



### 适用场景

- **数据量不大**（千以内）
- **空间**比时间**重要**
- 需要使用`Map`，且`key`为`int`类型。

### 源码分析

#### 构造函数

```java
    //用于标记value数组，作为已经删除的标记
    private static final Object DELETED = new Object();
    //是否需要GC 
    private boolean mGarbage = false;
    
    //存储key 的数组
    private int[] mKeys;
    //存储value 的数组
    private Object[] mValues;
    //集合大小
    private int mSize;
    
    //默认构造函数，初始化容量为10
    public SparseArray() {
        this(10);
    }
    //指定初始容量
    public SparseArray(int initialCapacity) {
        //初始容量为0的话，就赋值两个轻量级的引用
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
        //初始化对应长度的数组
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int[mValues.length];
        }
        //集合大小为0
        mSize = 0;
    }
```



#### 增 、改

**单个增、改：**

1、**二分查找法**查找key对应keys数组的索引，若找到则返回索引，若找不到则返回**低位去反**

2、如返回索引为正数，则直接替换mValues数组的该位置的值，否则，先对返回的i取反，得到应该插入的位置i，经过越界判断和该位置元素是否已删除判断，

将 mKeys[i] = key ，mValues[i] = value; 设置相应的值

3、GC 内存回收，for 循环 判断当前位置是否是已删除标记，如果是，则压缩keys、values数组（将已删除元素后面的元素移动到该位置）

2、**扩容时**，当前容量小于等于4，则扩容后容量为8.否则为**当前容量的两倍**。和`ArrayList,ArrayMap`不同(扩容一半)，和`Vector`相同(扩容一倍)。

```java
    public void put(int key, E value) {
        //利用二分查找，找到 待插入key 的 下标index
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        //如果返回的index是正数，说明之前这个key存在，直接覆盖value即可
        if (i >= 0) {
            mValues[i] = value;
        } else {
            //若返回的index是负数，说明 key不存在.
            
            //先对返回的i取反，得到应该插入的位置i
            i = ~i;
            //如果i没有越界，且对应位置是已删除的标记，则复用这个空间
            if (i < mSize && mValues[i] == DELETED) {
            //赋值后，返回
                mKeys[i] = key;
                mValues[i] = value;
                return;
            }
            
            //如果需要GC，且需要扩容
            if (mGarbage && mSize >= mKeys.length) {
                //先触发GC
                gc();
                //gc后，下标i可能发生变化，所以再次用二分查找找到应该插入的位置i
                // Search again because indices may have changed.
                i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
            }
            //插入key（可能需要扩容）
            mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
            //插入value（可能需要扩容）
            mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
            //集合大小递增
            mSize++;
        }
    }
    //二分查找 基础知识不再详解
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            //关注一下高效位运算
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        //若没找到，则lo是value应该插入的位置，是一个正数。对这个正数去反，返回负数回去
        return ~lo;  // value not present
    }
    
    //垃圾回收函数,压缩数组
    private void gc() {
        //保存GC前的集合大小
        int n = mSize;
        //既是下标index，又是GC后的集合大小
        int o = 0;
        int[] keys = mKeys;
        Object[] values = mValues;
        //遍历values集合，以下算法 意义为 从values数组中，删除所有值为DELETED的元素
        for (int i = 0; i < n; i++) {
            Object val = values[i];
            //如果当前value 没有被标记为已删除
            if (val != DELETED) {
                //压缩keys、values数组
                if (i != o) {
                    keys[o] = keys[i];
                    values[o] = val;
                    //并将当前元素置空，防止内存泄漏
                    values[i] = null;
                }
                //递增o
                o++;
            }
        }
        //修改 标识，不需要GC
        mGarbage = false;
        //更新集合大小
        mSize = o;
    }
```

**二分查找法**

表中元素是按升序排列，将表中间位置记录的[关键字](https://baike.baidu.com/item/关键字)与查找关键字比较，如果两者相等，则查找成功；否则利用中间位置[记录](https://baike.baidu.com/item/记录/1837758)将表分成前、后两个子表，如果中间位置记录的关键字大于查找关键字，则进一步查找前一子表，否则进一步查找后一子表。重复以上过程，直到找到满足条件的[记录](https://baike.baidu.com/item/记录/1837758)，使查找成功，或直到子表不存在为止，此时查找不成功。

简单示例如下

```java
int binarySearch(int[] nums, int target) {
    int left = 0, right = ...;

    while(...) {
        int mid = (right + left) / 2;
        if (nums[mid] == target) {
            ...
        } else if (nums[mid] < target) {
            left = ...
        } else if (nums[mid] > target) {
            right = ...
        }
    }
    return ...;
}
```



示意图

![put流程图](https://github.com/LiLustre/MyDevStudy/blob/master/%E5%9B%BE%E7%89%87/%E4%BA%8C%E5%88%86%E6%9F%A5%E6%89%BE%E6%B3%95.jpg?raw=true)

### 删除

1、按照key删除 根据key 利用二分查找法查找索引位置，然后将Value数组的i位置的元素设置为DELETED

2、然后将Value数组的i位置的元素设置为DELETED

```
    public void removeAt(int index) {
        //根据index直接索引到对应位置 执行删除操作
        if (mValues[index] != DELETED) {
            mValues[index] = DELETED;
            mGarbage = true;
        }
    }
```

```
    public void removeAtRange(int index, int size) {
    //越界修正
        final int end = Math.min(mSize, index + size);
        //for循环 执行单个删除操作
        for (int i = index; i < end; i++) {
            removeAt(i);
        }
    }
```



### 查

**按照key查询**

1、二分查找到 key 所在的index

2、返回Values数组位置的元素

```
    //按照key查询，如果key不存在，返回null
    public E get(int key) {
        return get(key, null);
    }

    //按照key查询，如果key不存在，返回valueIfKeyNotFound
    public E get(int key, E valueIfKeyNotFound) {
        //二分查找到 key 所在的index
        int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
        //不存在
        if (i < 0 || mValues[i] == DELETED) {
            return valueIfKeyNotFound;
        } else {//存在
            return (E) mValues[i];
        }
    }
```



**安装index查询**

1、按照下标查询时，需要考虑是否先GC

2、返回对应的key 或者Value

```
    public int keyAt(int index) {
    //按照下标查询时，需要考虑是否先GC
        if (mGarbage) {
            gc();
        }

        return mKeys[index];
    }
    
    public E valueAt(int index) {
     //按照下标查询时，需要考虑是否先GC
        if (mGarbage) {
            gc();
        }

        return (E) mValues[index];
    }
```

**查询下标：**

1、按照value查询下标时，不像key一样使用的二分查找。是直接线性遍历去比较，而且不像其他集合类使用equals比较，这里直接使用的 ==

2、如果有多个key 对应同一个value，则这里只会返回一个更靠前的index

```
    public int indexOfKey(int key) {
     //查询下标时，也需要考虑是否先GC
        if (mGarbage) {
            gc();
        }
        //二分查找返回 对应的下标 ,可能是负数
        return ContainerHelpers.binarySearch(mKeys, mSize, key);
    }
    public int indexOfValue(E value) {
     //查询下标时，也需要考虑是否先GC
        if (mGarbage) {
            gc();
        }
        //不像key一样使用的二分查找。是直接线性遍历去比较，而且不像其他集合类使用equals比较，这里直接使用的 ==
        //如果有多个key 对应同一个value，则这里只会返回一个更靠前的index
        for (int i = 0; i < mSize; i++)
            if (mValues[i] == value)
                return i;

        return -1;
    }
```





