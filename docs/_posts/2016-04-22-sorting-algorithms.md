---
layout: post
title: "常用的排序算法"
date: 2016-04-22
categories:
  - 算法
tags:
  - 排序
---

还记得上数据结构的时候，耗费了大量脑细胞却总是记不住种类繁多的排序算法；而勉强记住，应付了考试后，过了几个月又忘得一干二净，没办法，只能写下来供以后参考。

其实从排序算法的类别繁多以及在数据结构课程中占的时间比例可以看出这是计算机领域最经典也是研究最广泛的算法，这样说来，我等凡夫俗子不能马上顿悟也是情有可原了。

闲话少说，进入正题。

<!-- more -->

## 排序算法的应用领域

在深入算法之前，还是要强调一下排序算法之所以重要，在于它是构成其它算法的基础。比如下面就是应用了排序算法的例子：

* **搜索**： 排过序的数据可以使用二分查找（折半查找）快速地搜索某个元素。
* **最邻近对**： 给定n个数的集合，如何找到相差最小的一对数？如果集合已经排过序，那么一次线性查找就可以完成任务。
* **元素唯一性** -- 给定n个数的集合，里面有重复的元素吗？这是最邻近对问题的一个特例。
* **频率分布**： 给定n个数的集合，哪个数出现的次数最多？（求众数）。如果集合是有序的，相同的元素势必会连在一起，一次线性循环即可搞定。如果要查找任意一个元素k出现了多少次，首先用二分查找找到k，然后再往左移动，直到出现不是k的元素位置，同理也往右移动，这样便可得到k出现的次数。
* **选择**： 一个数列里第k大的元素是哪个？如果元素已经是有序的，那么第k个元素就是我们要找的。
* **找到两个集合的交集**： 如果对两个集合分别排序，两个集合是否相交以及交集是多少就很容易求了。

## 排序问题的定义
首先我们明确一下排序算法要解决的问题，也就是排序问题的定义。
算法都有输入和输出，排序问题的输入就是一个包含N条记录的数列，为了下文叙述方便，姑且称为数列A；而输出则是一个排好序的新的数列，这里的排序可以认为是从小到大的一个序列。

OK，让我们看看各种算法是怎样巧妙地把无序的数列变成有序的吧。

## 基本思路
虽然排序算法种类繁多，但是观其内在，都会使用**循环不变性**这种逻辑方式来解决问题。**循环不变性**的概念用这样的方式来处理排序问题：
* 从数列A的第一条记录遍历到最后一条记录，假设数列A有N条记录，那么转换成程序语言也就是从数组0索引循环至N-1索引
* 在循环的每一步中，都维持一个排好序的数列。这是循环不变性的直接体现。
* 每一步循环使得未排序的记录减1，排好序的数列中的记录加1。这样等到循环到最后的时候，整个数列就完成了排序。
* 不同排序算法的差别其实就在于如何遍历原数列以及如何维持排好序的数列。

## 选择排序

选择排序的基本思想是“选择”，即从未排序数列中，每次选出其中最小的元素到已排序的数列中。
下面我们就使用**循环不变性**的概念来组织一下选择排序的基本思路：

1. 输入：包含N条记录的数列A
2. 指针从 0 循环至 N-1
3. 指针为i时，找到剩下记录（也就是指针右侧）中最小的记录
4. 交换指针所处记录和找到的最小记录

在上面的思路中，只有一个数列A，而实际上，数列A的前半部是已排序的数列，而后半部是未排序的数列，中间的分隔则是当前循环的指针i所在位置。由此可见，随着循环不断前进，指针不断前移，已排序的数列越来越长，未排序的数列越来越短，到最后数列A就全部是已排序的了。从这里可以看出，我们只使用数列A的空间就完成了排序，这种只利用输入的数组，不需要额外的空间的称为原地排序。实际应用中，数列A可能惊人的大，因此能够原地排序是一个很好的特性。

下面是C++的实现代码，仅供参考

```c++
template<typename T>
void selection_sort(std::vector<T>& a, int lo, int hi) {
    // 从第一个记录遍历到最后一个记录
    for (int i = lo; i <= hi; ++i) {
        // 记录当前指针的位置
        int min_index = i;

        // 从剩下的记录中找到最小的一个来
        for (int j = i + 1; j <= hi; ++j) {
            if (a[min_index] > a[j]) {
                min_index = j;
            }
        }

        // 交换指针所处记录和找到的最小记录
        exch(a[i], a[min_index]);
    }
}
```

选择排序的基本特点

* 运行时间和输入没有关系，即使输入已经是有序的，也需要二次的时间复杂度
* 选择排序的记录移动是所有算法中最小的，每一个循环只有一次交换操作，总共只需要N次交换，也即N次移动


## 插入排序
在选择排序中，循环的每一步我们都要线性查找剩余记录中最小的一个记录，而找到记录之后只需一次交换就可以把找到的记录安置到已排序的数列中，即**选择**记录难，**插入**记录容易；插入排序的方式正好相反，即**插入**记录南，**选择**记录容易。插入排序每次选择的时候就按顺序从剩余记录中取一个，插入的时候得和已排好序的元素比较确定位置。但是由于要插入的是已排序的数列，因此相比于选择排序在无序数列中的选择，插入排序的插入会相对简单一些，特别是数列已经部分排序的时候。

另外插入排序还有一个特性，在插入排序中相同记录在排序前和排序后的相对顺序都是不变的，因为记录要改变位置需要和它左边的记录比较，而如果在排序前记录就大于左边的记录的话是不可能改变位置的。稳定排序通常在涉及到二次排序时比较有用，比如公司的人事数据库先按姓名把员工排好序，然后又按照年龄在之前排好序的基础之上继续排序，这时稳定排序保证年龄相同的员工的排序和第一次按姓名排序的顺序保持一致。

基本思想

1. 输入：包含N条记录的数列A
2. 指针从 0 循环至 N-1
3. 指针为i时，指针所在的记录和位于它左边并且比它大的记录交换

c++参考代码

```c++
    template<typename T>
    void insertion_sort(std::vector<T>& a, int lo, int hi) {
        // 从第一个记录遍历到最后一个记录
        for (int i = lo; i <= hi; ++i) {
            // 指针所在的记录和位于它左边并且比它大的记录交换
            for (int j = i; j > lo; --j) {
                if (a[j] < a[j - 1]) {
                    exch(a[j], a[j - 1]);
                } else {
                    break;
                }
            }
        }
    }
```

基本特点

* 如果输入记录是部分排序的话，插入排序的运行时间是线性的
* 插入排序是稳定的，选择排序可不是稳定的

## shell 排序

shell 排序是插入排序的增强版：插入排序在往左比较大小时每次只后退一步，而shell排序每次会后退多步。假设后退h步，那么得到的序列就是一个以h为间隔排序好的序列。shell排序会进行多次h排序。h的取值则从大慢慢减为1，这样做的理由是：

* 当h很大时，以h为间隔的序列较小，排序可以很快完成
* 当h变为1时，由于之前已经进行了多次h间隔排序，序列已经部分排序，因此插入排序此时的运行时间是线性的，也可以很快完成。

在实际中，通常h的取值序列有多种，不同的序列会导致不一样的时间复杂度。比较容易计算的是使用 3x + 1 这个公式来产生序列

 c++参考代码

```c++
    template<typename T>
    void shell_sort(std::vector<T>& a, int lo, int hi) {
        int N = hi - lo + 1;

        // decide decrease sequence
        int h = 1;
        while (h < N / 3) {
            h = 3 * h + 1;
        }

        while (h >= 1) {
            // h-sort the array
            for (int i = h + lo; i <= hi; ++i) {
                for (int j = i; j >= h + lo && a[j] < a[j - h]; j -= h) {
                    exch(a[j], a[j - h]);
                }
            }
            h = h / 3;
        }
    }
```

基本特点

* 实现代码少，常用于嵌入式编程
* 时间复杂度未知，但优于二次

## 堆排序

堆排序体现了好的数据结构对算法的帮助。堆排序和选择排序的原理一致，都是从剩下的记录中不断选择最小的记录出来。但是选择排序需要线性的时间去查找最小记录。而从一个集合中选择最小的记录出来是一个经典的优先队列解决的问题，如果使用堆或者平衡二叉树来实现优先队列的话，就能让这个操作变成$log(N)$时间。从而，借助更好的优先队列实现，堆排序把选择排序从$O(n^2)$复杂度提升到了$O(n*log(n))$

下面就详细介绍一下堆排序用到的数据结构
### 堆

堆是实现优先队列插入和获取最小值操作的简单而高效的数据结构。堆通过维持记录部分排序而非完全排序来工作，因此会比较高效。一个堆实际上可以用一个二叉树来表示（注意不是二叉搜索树）。在一个最小堆中，一个节点的键值总是比它的子节点要小；在一个最大堆中，一个节点的键值总是比它的子节点要大。

堆使用数组来实现，不需要使用任何的指针。键值在堆中的位置充当了指针的作用。在这个数组中，我们把二叉树的根节点存储在数组的第一个位置（为了方便，数组索引从1开始），相应地把它的左右两个子节点放在第二和第三的位置。一般地，我们可以把完全二叉数第$l$层的$2^l$个键值从左到右放在$2^{l-1}$和$2^l - 1$之间。并且节点之间有以下关系：

* 位于位置$k$的结点的父结点的位置是 $k / 2$
* 位于位置$k$的结点的子节点的位置是 $2k$ 和 $2k + 1$

### 如何构造一个堆
可以通过往数组末端不断插入记录来递增地构造一个堆。在插入新记录时，堆的顺序可能会不满足预定的条件：在最小堆中新记录可能小于它的父节点，或者是在最大堆中新纪录大于它的父节点。在这种情况下，需要交换这个记录和它的父节点的位置，这称作一次上游,对这个记录不断上浮直到不能继续上游为止，就维持了堆的既有顺序。下面的代码显示了最小堆的上游代码：

```c++
        void swim(int k) {
            // parent of node at k is k/2
            while (k > 1 && pq_[k / 2] > pq_[k]) {
                // if children's node is larger than parent, exchange
                exch(pq_[k], pq_[k / 2]);

                // swim up a level
                k /= 2;
            }
        }
```

对于一个有$n$个记录的堆来说，一次上浮最多只需要$lg(n)$次操作，因此，构造堆的时间复杂度为$O(n*log(n))$复杂度

### 如何从堆中取得最小值
从最小堆中取得最小的记录只需取数组的第一个元素即可，但是取完后二叉树会出现一个洞，需要把数组最后的一个记录填补到已经移除的第一个记录上；把最后一个记录移上来后可能会破坏堆的性质，如最小堆中根结点的记录可能会大于子结点，如果出现这种情况，需要将根结点和其较大的子结点交换，这称为一次下沉。下面是最小堆的下沉代码：

```c++
        void sink(int k) {
            // make sure k is not the bottom level
            while (2 * k <= N_) {
                // j is the left children
                int j = 2 * k;
                if (j < N_ && pq_[j] < pq_[j + 1]) {
                    // now, j is the bigger children
                    j++;
                }

                if (pq_[k] > pq_[j]) {
                    break;
                }

                // if parent node is smaller than the bigger children, exchange
                exch(pq_[k], pq_[j]);

                // sink down a level
                k = j;
            }

        }
```

对于一个有$n$个记录的堆来说，一次下沉最多只需要$lg(n)$次操作，因此，取得最小值的操作的时间复杂度为$O(log(n))$

### 更快的构建堆的方法

一条一条地插入记录来构造堆的方法需要$O(n*log(n))$的时间复杂度，如果记录序列全部已知，我们可以采用一种自底向上的构造方法，基本思路是从底端不是叶子结点的记录开始，做下沉操作，这样只需处理$n/2$个结点，这个时间复杂度基本上是线性的。下面是最大堆的下沉操作和构造方式：

```c++
  template<typename T>
    void sink(std::vector<T>& a, int k, int N) {
        // NOTE: the value of node k is a[k - 1]

        // make sure k is not the bottom level
        while (2 * k < N) {
            // j is the left children
            int j = 2 * k;
            if (j < N && a[j - 1] < a[j]) {
                // now, j is the bigger children
                j++;
            }

            if (a[k - 1] > a[j - 1]) {
                break;
            }

            // if parent node is smaller than the bigger children, exchange
            exch(a[k - 1], a[j - 1]);

            // sink down a level
            k = j;
        }
    }
    // Heap construction
    for (int k = N / 2; k >= 1; --k) {
        // loop for every non leaf node
        sink(pq, k, N);
    }
```

堆排序实现参考代码(这里用到了上面的最大堆的下沉方法)

```c++
    template<typename T>
    void heap_sort(std::vector<T>& pq) {
        int N = pq.size();

        // Heap construction
        for (int k = N / 2; k >= 1; --k) {
            // loop for every non leaf node
            sink(pq, k, N);
        }

        // Sort down
        while(N > 1) {
            exch(pq[0], pq[N - 1]);
            sink(pq, 1, --N);
        }
    }
```

基本特点

* 最坏的情况下也能达到$O(n*log(n))$，这是排序算法的理论最优。
* 缺点在于内部循环较长，无法使用缓存，并且是不稳定的，在实际中并不是最快的

## 归并排序

归并排序体现了分治的策略。主要思想是把大问题分解成小问题，不断递归去求解。下面的算法把数列递归地进行分解，然后再合并。

代码实现

```c++
    // merge tow subarray
    template<typename T>
    void merge(std::vector<T>& a, std::vector<T>& aux, int lo, int mid, int hi) {
        for (int i = lo; i <= hi; ++i) {
            aux[i] = a[i];
        }

        int m = lo;
        int n = mid + 1;
        for (int i = lo; i <= hi; ++i) {
            if (m > mid) {
                a[i] = aux[n++];
            } else if (n > hi) {
                a[i] = aux[m++];
            } else if (aux[n] < aux[m]) {
                a[i] = aux[n++];
            } else {
                a[i] = aux[m++];
            }
        }
    }

    // resuive sort
    const int CUTOFF = 7;
    template<typename T>
    void merge_sort(std::vector<T>& a, std::vector<T>& aux, int lo, int hi) {
        if (hi <= lo) {
            return;
        }

        // use insertion sort for small subarrays
        if (hi <= lo + CUTOFF - 1) {
            insertion_sort(a, lo, hi);
            return;
        }

        int mid = lo + (hi - lo) / 2;
        merge_sort(a, aux, lo, mid);
        merge_sort(a, aux, mid + 1, hi);

        // do not merge if already sorted
        if (a[mid] < a[mid + 1]) {
            return;
        }

        merge(a, aux, lo, mid, hi);
    }
```

上述实现中借助了一个额外的aux数组来存储记录，并且在子问题规模很小时采用了插入排序。

基本特点

* 归并排序的平均时间复杂度为$O(n*log(n))$
* 归并排序不是原地排序，需要额外的存储空间


## 归并排序的非递归实现

基本思想

1. 遍历数组，首先归并排序大小为1的子数组
2. 继续遍历，不断归并大小为2，4，16的子数组

```c++
    template<typename T>
    void bottom_up_merge_sort(std::vector<T>& a, std::vector<T>& aux, int lo, int hi) {
        int N = hi - lo + 1;
        for (int sz = 1; sz < N; sz += sz) {
            for (int k = lo; k < lo + N - sz; k += (sz + sz)) {
                merge(a, aux, k, k + sz - 1, std::min(k + sz + sz - 1, N - 1));
            }
        }
    }
```

基本特点

* 如果有足够的空间的话，非递归的归并排序的稳定性是工业级别的

## 快速排序

快速排序和归并排序类似，都是递归的算法，通过把问题分解为子问题来解决。不同的是，归并排序每次都把问题分成相同大小的两个子问题，然后通过归并操作进行合并；而快速排序则通过拆分的方式来分解问题，即每次找一个中间元素，把记录分成小于该中间元素（在中间元素左边）和大于该中间元素（在中间元素右边）的这两部分，此时中间元素已经排好序，只需对左右两边递归继续采用相同方式拆分即可。

和归并排序的归并操作是线性的时间复杂度类似，快速排序的拆分操作也是线性的。归并排序和快速排序的递归分解都把问题变成了一个二叉树的结构，而归并排序的二叉树是完全二叉树，因此树高是$lg(n)$，而快速排序的树高则与中间元素的选取有很大的关系，为了达到了归并排序相似的树高，要求输入记录必须是无序的，研究表明，无序的二叉树插入的树高平均只比完全二叉树高36%，因此该种情况下的快速排序和归并排序的时间复杂度是相同的。当然由于快速排序加入了随机的因素，我们只能说平均情况下快速排序和归并排序的时间复杂度是相同的，也不排除很小的概率的情况下快速排序的时间复杂度为$n^2$

基本思想

* 随机打乱原始记录
* 针对索引为j的记录进行拆分，使得：
  * 记录a[j]位于最终已排序的位置
  * j左边的记录没有比a[j]大的
  * j右边的记录没有比a[j]小的
* 对拆分后的各个部分递归进行上述处理

代码实现

```c++
    template<typename T>
    int partition(std::vector<T>& a, int lo, int hi) {
        int i = lo;
        int j = hi + 1;

        while(true) {
            // process i pointer
            // find item on left to swap
            while(a[++i] < a[lo]) {
                if (i == hi) {
                    break;
                }
            }

            // process j pointer
            // find item on right to swap
            while(a[--j] > a[lo]) {
                if (j == lo) {
                    break;
                }
            }

            // find if pointers cross
            if (i >= j) {
                break;
            }

            // swap
            exch(a[i], a[j]);
        }

        // swap with partition item
        exch(a[lo], a[j]);

        // return index of item now known to be in place
        return j;
    }


    template<typename T>
    void quick_sort_sub(std::vector<T>& a, int lo, int hi) {
        if (hi <= lo + CUTOFF) {
            // improvement 1:  use insertion fort for small subarray
            insertion_sort(a, lo, hi);
            return;
        }

        // improvement 2: estimate partition item with median of three samples
        int m = median_of_three(a, lo, lo + (hi - lo)/ 2, hi);
        exch(a[lo], a[m]);

        int j = partition(a, lo, hi);
        quick_sort_sub(a, lo, j - 1);
        quick_sort_sub(a, j + 1, hi);
    }

    template<typename T>
    void quick_sort(std::vector<T>& a) {
        // shuffle is needed for performance guarantee
        shuffle(a);

        quick_sort_sub<T>(a, 0, static_cast<int>(a.size()) - 1);
    }
```

上面的代码使用了两个提升：和归并排序中一样，我们在记录序列很小时采用了插入排序；另外我们本来是采用随机打乱后的记录顺序来选取中间值，为了让得到的二叉树更加平衡，我们需要选择接近数列中位数的记录作为中间值，这里我们采用了抽样的方式来计算中值。

基本特点

虽然快速排序理论上只能在概率上趋近于$n*lg(n)$的时间复杂度，但是由于它的内层循环较小，并且容易利用计算机缓存等原因，一个设计得很好的快速排序的效率是归并排序和堆排序的2-3倍！

在实际应用中，如果记录有许多重复的话，会发现快速排序接近于$n^2$的时间复杂度，这时候我们需要使用快速排序的改进版：3路快速排序

## 3路快速排序

基本思想是：

* 把记录序列查分成3部分（而不是之前的两部分）
* 在lt和gt中间的记录都等于中间元素
* lt左边的记录都不大于中间元素
* lt右边的记录都不小于中间元素

实现代码

```c++
    // * Let v be partitioning item a[lo].
    // * Scan i from left to right.
    //  - (a[i] < v): exchange a[lt] with a[i]; increment both lt and i
    //  - (a[i] > v): exchange a[gt] with a[i]; decrement gt;
    //  - (a[i] == v): increment i
    template<typename T>
    void quick_sort_3way_sub(std::vector<T>& a, int lo, int hi) {
        if (hi <= lo + CUTOFF) {
            // improvement 1:  use insertion fort for small subarray
            insertion_sort(a, lo, hi);
            return;
        }

        int lt = lo;
        int i = lo;
        int gt = hi;

        // improvement 2: estimate partition item with median of three samples
        int m = median_of_three(a, lo, lo + (hi - lo)/ 2, hi);
        exch(a[lo], a[m]);

        // partition item
        T v = a[lo];

        while(i <= gt) {
            if (a[i] < v) {
                exch(a[lt++], a[i++]);
            } else if (a[i] > v) {
                exch(a[i], a[gt--]);
            } else {
                i++;
            }
        }

        quick_sort_3way_sub(a, lo, lt - 1);
        quick_sort_3way_sub(a, gt + 1, hi);
    }

    template<typename T>
    void quick_sort_3way(std::vector<T>& a) {
        // shuffle is needed for performance guarantee
        shuffle(a);

        quick_sort_3way_sub<T>(a, 0, static_cast<int>(a.size()) - 1);
    }
```

至此，经典的排序方法已经介绍完毕。除了选择排序和插入排序需要二次的时间复杂度外，堆排序、归并排序以及快速排序都能达到$n*lg(n)$的时间复杂度，而这也是证明了的排序算法时间复杂度的下界，即这已经是最优算法了。但是从之前的讨论可以看到，在实际情况中，受到各种因素的限制，时间复杂度相同的算法的实际效率并不同，并且有可能相差数倍，当然，这是大O方式来衡量时间复杂度的一个弊端：即它只能忽略影响算法效率的其它因素，单单从输入规模上来看算法运行时间随输入规模的变化。从这个角度来看，虽然堆排序、归并排序以及快速排序都是最优算法，但是还可能有更快的排序算法等待着我们去发掘。

## 几种排序方法的比较

最后再整体比较一下上述的几类排序算法，算是一个总结：

排序方法 | 原地排序 | 稳定排序 | 最差 | 平均 | 最好 | 备注
--- | --- | --- | --- | --- | --- | ---
选择排序 | 是 | 否 | $$N^2/2$$ | $$N^2/2$$ | $$N^2/2$$ | 只需要N次交换
插入排序 | 是 | 是 | $$N^2/2$$ | $$N^2/4$$ | $$N$$ | N很小或数据部分排序时适用
shell排序 | 是 | 否 | 未知 | 未知 | $$N$$ | 实现代码少常用于嵌入式编程，时间复杂度未知但低于二次 |
快速排序 | 是 | 否 | $$N^2/2$$ | $$2N*lg(N)$$ | $$lg(N)$$ | 实际使用中最快的算法
3路快速排序 | 是 | 否 | $$N^2/2$$ | $$2N*lg(N)$$ | $$N$$ | 快速排序在大量重复记录下的改进
归并排序 | 否 | 是 | $$N*lg(N)$$ | $$N*lg(N)$$ | $$N*lg(N)$$ | 稳定排序
堆排序 | 是 | 否 | $$2N*lg(N)$$ | $$2N*lg(N)$$ | $$N*lg(N)$$ | 原地排序的（节省空间）
？？？ | 是 | 是 | $$N*lg(N)$$ | $$N*lg(N)$$ | $$N*lg(N)$$ | 终极排序算法:)
