---
layout:     post
title:      "排序总结"
date:       2019-10-20 16:20:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - 算法

---

[![KKrSaD.png](https://s2.ax1x.com/2019/10/20/KKrSaD.png)](https://imgchr.com/i/KKrSaD)


[![yeHdm9.png](https://s3.ax1x.com/2021/02/01/yeHdm9.png)](https://imgchr.com/i/yeHdm9)


<br><br>

## 导航
[一. Bubble Sort](#jump1)
<br>
[二. Selection Sort](#jump2)
<br>
[三. Insertion Sort](#jump3)
<br>
[四. Quick Sort](#jump4)
<br>
[五. Merge Sort](#jump5)


<br><br>
## <span id="jump1">一. Bubble Sort</span>

```
/**
 * @author: simab.hjf
 * @description: 冒泡排序
 * @date: Created in  2021-02-01 21:58
 */
public class BubbleSort {
    public static void bubbleSort(int[] arr) {
        if (arr == null) {
            return;
        }

        for (int i = arr.length - 1; i > 0; i--) {
            for (int j = 1; j <= i; j++) {
                if (arr[j - 1] > arr[j]) {
                    int tmp = arr[j - 1];
                    arr[j - 1] = arr[j];
                    arr[j] = tmp;
                }
            }
        }
    }
}
```

算法描述: 从数组第一个元素开始,依次比较其与后面相邻元素的大小,若前一元素小于后一元素,则交换,否则不变.第一轮遍历交换完成,则数组中最大的元素就已经被挪动到了数组最后的位置,形如冒泡的过程,因此称冒泡排序.<br>

外层循环 i ,标记冒泡比较需要执行到的最后一位数组下标位置,随着轮次的不断进行,该下标不断前移.这里 i > 0 或 i >= 0 均可,因为当 i = 1 时,表示数组中除了最小的元素外,其他元素均已被挪动到了正确位置,那么因此剩下的那个元素也一定处于了正确位置中了,而且当 i = 0 的时候,可以发现内层循环已不再需要执行了.<br>

内层循环 j ,就是控制前后两个元素的比较,若 j 从 1 开始,那么就与它的前一个元素比较,因此结束于 i ; j 也可以从 0 开始,那么就与它的后一个元素比较,此时就要结束与 i - 1 . <br>



<br><br>
## <span id="jump2">二. Selection Sort</span>

```
/**
 * @author: simab.hjf
 * @description: 选择排序
 * @date: Created in  2021-02-01 22:33
 */
public class SelectionSort {
    public static void selectionSort(int[] arr) {
        if (arr == null) {
            return;
        }
        for (int i = 0; i < arr.length - 1; i++) {
            int flagIdx = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[flagIdx] > arr[j]) {
                    flagIdx = j;
                }
            }
            int tmp = arr[flagIdx];
            arr[flagIdx] = arr[i];
            arr[i] = tmp;
        }
    }
}
```

算法描述: 每一轮次遍历,选出待比较区间内最小的那个元素,与当前待比较区间的第一个元素交换,那么此时该元素就处于正确的位置已被排好序了.循环执行以完成全部排序操作.例如,第一轮循环选出了最小的元素,放到第一个位置,此时待比较区间的起点后移一格(因为第一个位置元素已经排好了);接下来第二轮循环选出第二小的放在第二个位置,以此类推.<br>

外层循环 i ,标记当前轮次比较后要正确放置最小元素的下标.当数组中前 arr.length - 1 的元素已经被正确放置时,剩下的最后一个元素一定也已经处在正确位置上了,因此可以在 arr.length - 1 处结束.当然,在 arr.length 处结束也没问题,内层循环不再满足条件会直接跳过.<br>

flagIdx 记录该轮遍历过程中,最小元素的下标.当然也可以不记,而采用比较过程中每找到小于起始位置元素值的项,就与起始位置元素交换,但这样多了很多次交换操作.而记录 flagIdx 下标<br>

内层循环 j , 控制需要与当前最小元素进行比较的下标,每轮都比较到最后.<br>

一轮比较完成后,进行一次交换,这时候也有可能起始元素就是最小的,此时即使交换也没有影响(自己和自己交换).<br>


<br><br>
## <span id="jump3">三. Insertion Sort</span>

```
/**
 * @author: simab.hjf
 * @description: 插入排序
 * @date: Created in  2021-02-01 22:56
 */
public class InsertionSort {

    public static void insertionSort(int[] arr) {
        if (arr == null || arr.length == 0) {
            return;
        }
        for (int i = 0; i < arr.length - 1; i++) {
            int j = i + 1;
            int needInsertionEleValue = arr[j];
            for (; j > 0; j--) {
                if (needInsertionEleValue < arr[j - 1]) {
                    arr[j] = arr[j - 1];
                } else {
                    break;
                }
            }
            arr[j] = needInsertionEleValue;
        }
    }
}
```

算法描述: 通过构建有序序列,对于未排序数据,在已排序序列中从后向前扫描,找到相应位置并插入.一般来说,插入排序都采用in-place在数组上实现,具体如下:
* 步骤1: 从第一个元素开始,该元素可以认为已经被排序;
* 步骤2: 取出下一个元素,在已经排序的元素序列中从后向前扫描;
* 步骤3: 如果该元素(已排序)大于新元素,将该元素移到下一位置;
* 步骤4: 重复步骤3,直到找到已排序的元素小于或者等于新元素的位置;
* 步骤5: 将新元素插入到该位置后;
* 步骤6: 重复步骤2~5.



<br><br>
## <span id="jump4">四. Quick Sort</span>

```
/**
 * @author: simab.hjf
 * @description: 快速排序
 * @date: Created in  2021-02-01 23:41
 */
public class QuickSort {
    public static void quickSort(int[] arr, int left, int right) {
        if (arr == null || arr.length == 0 || arr.length == 1) {
            return;
        }
        if (left < right) { //重要
            int benchmarkIdx = divide(arr, left, right);
            quickSort(arr, left, benchmarkIdx - 1);
            quickSort(arr, benchmarkIdx + 1, right);
        }

    }

    public static int divide(int[] arr, int left, int right) {
        int benchmark = arr[left];
        while (left < right) {
            while (benchmark <= arr[right] && left < right) {
                right--;
            }
            arr[left] = arr[right];

            while (benchmark >= arr[left] && left < right) {
                left++;
            }
            arr[right] = arr[left];
        }
        arr[right] = benchmark;
        return right;
    }
}
```

算法描述: 从数组中选择一个元素,把这个元素称之为中轴元素,然后把数组中所有小于中轴元素的元素放在其左边,所有大于或等于中轴元素的元素放在其右边,显然,此时中轴元素所处的位置的是其最终的正确位置.然后递归对前后两个部分执行相同操作,最终数组整体有序.


<br><br>
## <span id="jump5">五. Merge Sort</span>

```
/**
 * @author: simab.hjf
 * @description: 归并排序
 * @date: Created in  2021/2/6 上午10:47
 */
public class MergeSort {
    public static void mergeSort(int[] arr, int left, int ritht) {
        if (arr == null || arr.length == 0 || left == ritht) {
            return;
        }
        int middle = (left + ritht) / 2;
        mergeSort(arr, left, middle);
        mergeSort(arr, middle + 1, ritht);

        int[] auxiliaryArr = new int[ritht - left + 1];
        int auxiliartArrIdx = 0;
        int leftArrIdx = left;
        int rithtArrIdx = middle + 1;

        while (leftArrIdx <= middle && rithtArrIdx <= ritht) {
            if (arr[leftArrIdx] <= arr[rithtArrIdx]) {
                auxiliaryArr[auxiliartArrIdx++] = arr[leftArrIdx++];
            } else {
                auxiliaryArr[auxiliartArrIdx++] = arr[rithtArrIdx++];
            }
        }
        while (leftArrIdx <= middle) {
            auxiliaryArr[auxiliartArrIdx++] = arr[leftArrIdx++];
        }

        while (rithtArrIdx <= ritht) {
            auxiliaryArr[auxiliartArrIdx++] = arr[rithtArrIdx++];
        }
        for (int i = left; i <= ritht; i++) {
            arr[i] = auxiliaryArr[i - left];
        }
    }
}
```

算法描述: 将一个大的无序数组排序,可以先把大的数组分成两个,然后对这两个数组分别进行排序,之后在把这两个数组合并成一个有序的数组.分治思想.
* 步骤1: 把长度为n的输入序列分成两个长度为n/2的子序列;
* 步骤2: 对这两个子序列分别采用归并排序;
* 步骤3: 将两个排序好的子序列合并成一个最终的排序序列