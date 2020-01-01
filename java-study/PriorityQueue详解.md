### 一、基本介绍
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;优先队列PriorityQueue是Java的一种队列，作为队列它也是基于Queue接口实现，有入队(add、offer)和出队(remove、poll)操作。PriorityQueue通过小顶堆实现，保证每次取出的元素都是队列中权重值最小的，权重值的判断基于元素自身的compareTo方法(需要元素的类实现Comparable接口的compareTo方法)，或者可以自己构造Comparator传入作为比较器。

PriorityQueue的构造方法如下:

```java
//无自定义Comparator
public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
}
//自定义Comparator
public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
}
```

### 二、数据结构
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PriorityQueue底层是基于小顶堆实现，而小顶堆是一种经过排序的完全二叉树，其中任一非终端节点的数据值均不大于其左子节点和右子节点的值。既然小顶堆是一颗完全二叉树，那么就可以用一个一维数组来存储树的节点，PriorityQueue类中用定义了queue这个变量来存储元素,size变量来表示数组大小：

```java
    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    transient Object[] queue; // non-private to simplify nested class access

    /**
     * The number of elements in the priority queue.
     */
    private int size = 0;
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于是完全二叉树，那么父子节点的位置关系也很好计算，设NodePos为当前节点数组下标位，parentPos为它的父节点数组下标位，leftPos和rightPos为父节点的左右子节点数组下标位，为它们的关系如下：

```
parentPos = (nodePos - 1)/2
leftPos = parentPos * 2 + 1
rigthPos = parentPos * 2 + 2
```

我们通过如下代码构造一个PriorityQueue队列，再通过示意图来更直观的了解内部的数据结构：

```java
int[] num = {3,10,5,16,1,19,9,13,20,12};
PriorityQueue<Integer> queue = new PriorityQueue<>();
for(int n : num){
    queue.offer(n);
}
```
元素在queue数组中排列如下：

![image](https://github.com/yangml1993/source-study/blob/master/img/20200101-1.png)

转化为树的表示形式如下图：

![image](https://github.com/yangml1993/source-study/blob/master/img/20200101-2.png)
### 三、基本操作
#### 2、入堆操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;入堆通过add(E e)方法或offer(E e)方法,两者语义相同，只是Queue接口规定二者对插入失败时的处理不同，前者在插入失败时抛出异常，后则则会返回false。入堆插入新的元素这个过程中，新的元素可能会破坏小顶堆的性质，所以需要做调整，PriorityQueue把这个过程叫做siftUp，并定义了siftUp(int k, E x)来实现，以offer方法为例：

```java
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
    /**
     * Inserts item x at position k, maintaining heap invariant by
     * promoting x up the tree until it is greater than or equal to
     * its parent, or is the root.
     *
     * To simplify and speed up coercions and comparisons. the
     * Comparable and Comparator versions are separated into different
     * methods that are otherwise identical. (Similarly for siftDown.)
     *
     * @param k the position to fill
     * @param x the item to insert
     */
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}
private void siftUpUsingComparator(int k, E x) {
    //遍历二叉树找到元素的插入位置
    while (k > 0) {
        int parent = (k - 1) >>> 1;//计算当前节点的父节点
        Object e = queue[parent];//父节点元素
        if (comparator.compare(x, (E) e) >= 0)
            break;//如果插入元素比父节点大，跳出循环
        queue[k] = e;//将父节点下移到当前节点
        k = parent;//改变k的值为父节点的位置
    }
    queue[k] = x;//插入元素
}
```
siftUp的过程是，从k=size + 1开始，比较k的父节点值和插入元素值，如果父节点值较大，则将父节点下移到k位置，接着令k=parent，再和上一个父节点比较，直到父节点值小于插入元素值时，此时的k值就是元素的插入位置。我们以之前构造的PriorityQueue为例，插入一个值为2的元素，整个过程如下图所示：

![image](https://github.com/yangml1993/source-study/blob/master/img/20200101-3.png)
#### 3、出堆操作
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;出堆通过remove(E e)方法或poll(E e)方法,两者语义相同，只是Queue接口规定二者对插入失败时的处理不同，前者在出队失败时抛出异常，后则则会返回false。出堆是将树的根节点也就是数组的第一个元素从树中取出，由于根节点被取出，就必须要重新调整树的结构，以满足小顶堆的要求，这个过程叫siftDown，通过siftDown(int k, E x)来实现,以poll方法为例：

```java
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];//取根节点即数组第一位元素返回
        E x = (E) queue[s];//取最末尾的叶子节点即数组最后一位元素
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);//调用siftDown方法调整树
        return result;
    }
    /**
     * Inserts item x at position k, maintaining heap invariant by
     * demoting x down the tree repeatedly until it is less than or
     * equal to its children or is a leaf.
     *
     * @param k the position to fill
     * @param x the item to insert
     */
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }
    //k为0代表根节点，x为数组最后一个元素
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {//k从0也就是根节点开始遍历，找到两个子节点值较小的一个，往下遍历
            int child = (k << 1) + 1;//k所在节点的左子节点位置
            Object c = queue[child];//左子节点元素值
            int right = child + 1;//右子节点位置
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                //如果右子节点比左子节点小，则child改为right
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                //如果x小于child的值，找到插入位置，结束遍历
                break;
            queue[k] = c;//值较小的子节点上移到父节点位置
            k = child;//k值改为child，从child位继续开始遍历
        }
        queue[k] = x;//将x插入到k位
    }
```
siftDown的过程是，将最末尾的叶子节点(lastNode)取出，放到根节点位置，接着和两个子节点较小的那一个(childNode)比较，如果lastNode较大，则交换两个节点的位置，childNode上移到父节点位置，lastNode移到child位置。接着，lastNode继续向下比较，直到lastNode值小于childNode，将lastNode插入到当前位置，完成调整。我们以之前构造的PriorityQueue为例，取出根节点1，整个调整过程如下图所示：

![image](https://github.com/yangml1993/source-study/blob/master/img/20200101-4.png)