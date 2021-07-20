---
title: 经典算法之快选求TOPK
date: 2021-07-20 08:53:57
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "算法" #分类
tags:   #标签
    - 面试题
    - 算法
    - JAVA
---
#### 快选法求未排序数组中第KTH大的元素
题目是有一个未排序的数组，求里面第k大的元素，例如
``` java
  int[] nums = {2, 1, 4, 3, 6, 5, 9, 7, 8};
```
这个数组的元素是1-9，未排序。
k=1时，则返回9
k=2时，则返回8
k=9时，则返回1
可能会想到，首先对这个数组排序一下（从小到大），然后直接返回nums.length - k的元素即可。
或者从大到小排序，返回数组中位于k-1的元素也是可以的。
这里我们使用快选法来求解，快选法的原理是，使用快排的框架来求解。
快排的划分算法，一次划分的结果得到的pivot值，就是位于数组中pivot的位置，是确定的，我们对比
pivot与kth的值（kth=nums.length - k),如果pivot==kth则，我们直接返回nums[pivot]即可。
如果pivot 小于 kth 则我们下次划分的时候把左边界设为pivot + 1 即可；如果pivoit 大于 kth,
则我们把右边界设置为 pivot - 1，然后再次调用划分，如果最后找不到我们就返回-1。
因为前面我在文章中已经练习过快排和二分查找，所以这个时候练习这算法非常合适，比较容易接受。
这个算法刚好就结合了快排的partition划分和二分查找，可以使用递归和迭代两种实现。
时间复杂度：平均是O(n) 最坏情况下是O(N平方)
代码如下：
``` java

    static void swap(int[] nums, int i, int j ){
        int t = nums[i];
        nums[i] = nums[j];
        nums[j] = t;
    }

    static int partition(int[] nums, int left, int right) {
        int i = left;
        int j = left;
        while(j <= right) {
            if(nums[j] < nums[left]) swap(nums, j, ++i);
            j++;
        }
        swap(nums, i, left);
        return i;
    }

    static int quickSelectKth(int[] nums, int left, int right, int k){
        int kth = nums.length - k;
        while( left <= right) {
            int pivot = partition(nums, left, right);
            if(pivot == kth) return nums[kth];
            if(pivot > kth) right = pivot - 1;
            if(pivot < kth) left = pivot + 1;
        }
        return -1;
    }

    static int quickSelectKthR(int[] nums, int left, int right, int k){
        if(left >= right) return -1;
        int kth = nums.length - k;
        int pivot = partition(nums, left ,right);
        if(pivot == kth) return nums[kth];
        if(pivot > kth) return quickSelectKthR(nums, left, pivot -1 , k);
        if(pivot < kth) return quickSelectKthR(nums, pivot + 1, right, k);
        return -1;
    }
```