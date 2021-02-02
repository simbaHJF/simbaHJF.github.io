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


## 导航
[一. Bubble Sort](#jump1)
<br>
[二. Selection Sort](#jump2)
<br>




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
