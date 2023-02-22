# TopK的问题

> 个人认为算法的意义是，刷题百变其义自见。

TopK考察的点很多,可以聊的很多。这个不仅在业务场景中会大量应用,在算法的实现上也是千变万化。解答这个题不要使用奇技淫巧的方式,因为仅仅把算法题做出来,是没有什么卵用的,还是要深刻理解其中的变数做到灵活变化。

## 业务角度

举一个例子

像 Twitter 的 Trending Topic，微博热搜，视频网站的点击排行，下载排行（可以是日榜、月榜、总榜）等等。这样的系统，在统计数据非常大（heavy hitters）的时候，其中的挑战性在于两个：

1. 无法简单地在单台机器的内存中进行目标 id -> count 计数的简单映射，因为数据量太大，内存放不下。
2. 无法用实时的方式高效地显示出动态变化的 Top K 列表来。

## 算法角度

如何设计一个最优解去解决这个问题？

### 排序

任选其一的排序函数, 按照从大到小的顺序排序,选择第K-1的下标,既为结果

### 堆

设计一个size为K的最小堆结构

- 非常好的一种思路,维护一个最小堆

```java
     public int findKthLargest2(int[] nums, int k) {
        PriorityQueue<Integer> heap = new PriorityQueue<>();
        for (int num : nums) {
            heap.add(num);
            if (heap.size() > k) {
                heap.poll();
            }
        }
        return heap.peek();
    } 
```

### 快速排序(优化)

```java
  public int partition(int[] a, int left, int right) {
        //中心元素
        int pivot = a[right];
        int p = left;
        for (int i = left; i < right; i++) {
            if (a[i] <= pivot) {
                int temp = a[i];
                a[i] = a[p];
                a[p] = temp;
                p++;
            }
            System.out.println(Arrays.toString(a));
        }
        // 将中心元素和指针指向的元素交换位置
        int temp = a[p];
        a[p] = a[right];
        a[right] = temp;
        System.out.println(p);
        return p;
    }

    public void quickSort(int[] array, int low, int high) {
        if (low < high) {
            // 获取划分子数组的位置
            int position = partition(array, low, high);
            // 左子数组递归调用
            quickSort(array, low, position - 1);
            // 右子数组递归调用
            quickSort(array, position + 1, high);
        }
    }


    /**
     * 最大的K个数
     *
     * @param nums
     * @param k
     * @return
     */
    public int findKthLargest(int[] nums, int k) {
        int len = nums.length;
        int left = 0;
        int right = len - 1;
        // 转换一下，第 k 大元素的下标是 len - k
        int target = len - k;
        while (true) {
            int index = partition(nums, left, right);
            if (index == target) {
                return nums[index];
            } else if (index < target) {
                left = index + 1;
            } else {
                right = index - 1;
            }
        }
    }
```

## 系统设计

回头解答一下上面的两个实际应用中的问题, 

### 实时

Top的实时问题.以微博的热榜为例, 最热的话题一定得是评论最多,点赞最多, 转发最多等多维度聚合的结果。提及此,更有可能想到的是大数据的处理方式。

假如抛开大数据的处理模式, 如果只依靠DB是如何处理. 

存

首先对DB进行分表, 按照分钟拆分为60个表,每一分钟都存在一个表.

当前话题,每一次处理就hash%60,然后对key+1

#### 存不下了怎么办？

过期时间:  增加天的维度进行存储

两部分存储:  根据业务独立出一张表, 每次存的时候进行计算数据。

查

当前这一分钟的top,利用mysql的order by 跟 limit语句即可

- 延时

利用cacel订阅+MQ消费



蛇有蛇路 鼠有鼠道。因为我只考虑使用DB,所以会这么思考。同样的如果使用redis方案会更合适, 那redis就必然会带来问题点在与<缓存的不一致性。以及redis的部署方案的选择。选择一个合适的去解决问题即可.



发现一些还不错的文章,可以用下面的思路去接着想后面的问题

(参考一)

[TopK](https://github.com/thachlp/system-design-concept/blob/master/linkedin/topk.md)

(参考二)

[问一道经典系统设计题|一亩三分地系统设计版](https://www.1point3acres.com/bbs/thread-159538-1-1.html)
