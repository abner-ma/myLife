# 快速排序算法

## 介绍
快速排序算法被认为是冒泡排序算法的一种改进，是由C. A. R. Hoare在1962年提出。
它的基本思想是：通过一次排序将要排序的数据分割成独立的两部分，其中一部分的数据都比另一部分的所有数据都要小，
然后在按此方法对这两部分数据分别进行快速排序，整个过程可以递归进行，以此达到对一组数据的排序。

---

## 算法流程
设要排序的数组是A[0]到A[n-1]，首先任意选取一个数（通常选择数组的第一个数）作为关键数据，然后让所有比它小的数据都放到它前面，让所有比它大的数据都放在它后面，
快速排序是一种不稳定的排序算法。
一次排序的流程为：
1.设置两个变量i，j，排序开始的时候，i=0，j=n-1；
2.以第一个数组元素作为关键数据，赋值到零时变量key。key=A[0];
3.从j开始向前搜索，j--,找到第一个小于key的值，将A[i]与A[j]互换；
4.从开始向后搜索，i++找到第一个大于key的，将A[i]与A[j]互换；
5.重复第3、4步，直到i=j；
（3、4步中，没有找到符合条件的值，则i++，j--，直到找到为止）

---

## C语言快排函数qsort

在c语言中可以用函数qsort（）直接为数组进行排序，该函数包含在stdlib.h中 
用 法:
void qsort(void *base, int nelem, int width, int (*fcmp)(const void *,const void *));
参数：
　　1 待排序数组首地址
　　2 数组中待排序元素数量
　　3 各元素的占用空间大小
　　4 指向函数的指针，用于确定排序的顺序

代码：
```
#include <stdio.h>
#include <stdlib.h>
int cmp(const void *a,const void *b)
{
    return *(int *)a-*(int *)b;//这是从小到大排序，若是从大到小改成： return *(int *)b-*(int *)a;
}
int main()
{
    int a[100];
    int n;
    scanf("%d",&n);//n代表数组中有几个数字
    int i;
    for(i=1;i<=n;i++)
        scanf("%d",&a[i-1]);
    qsort(a,n,sizeof(a[0]),cmp);//(数组，需要排序的数字个数，单个数字所占内存大小，比较函数）
     for(i=1;i<=n;i++)
        printf("%d ",a[i-1]);
    return 0;
}
```

---

## C语言实现

代码：
```
void sort(int *a, int left, int right)
{
    if(left >= right)/*如果左边索引大于或者等于右边的索引就代表已经整理完成一个组了*/
    {
        return ;
    }
    int i = left;
    int j = right;
    int key = a[left];
     
    while(i < j)                               /*控制在当组内寻找一遍*/
    {
        while(i < j && key <= a[j])
        /*而寻找结束的条件就是，1，找到一个小于或者大于key的数（大于或小于取决于你想升
        序还是降序）2，没有符合条件1的，并且i与j的大小没有反转*/ 
        {
            j--;/*向前寻找*/
        }
         
        a[i] = a[j];
        /*找到一个这样的数后就把它赋给前面的被拿走的i的值（如果第一次循环且key是
        a[left]，那么就是给key）*/
         
        while(i < j && key >= a[i])
        /*这是i在当组内向前寻找，同上，不过注意与key的大小关系停止循环和上面相反，
        因为排序思想是把数往两边扔，所以左右两边的数大小与key的关系相反*/
        {
            i++;
        }
         
        a[j] = a[i];
    }
     
    a[i] = key;/*当在当组内找完一遍以后就把中间数key回归*/
    sort(a, left, i - 1);/*最后用同样的方式对分出来的左边的小组进行同上的做法*/
    sort(a, i + 1, right);/*用同样的方式对分出来的右边的小组进行同上的做法*/
                       /*当然最后可能会出现很多分左右，直到每一组的i = j 为止*/
}

```

---

## C++语言版
代码：

```
#include <iostream>
 
using namespace std;
 
void Qsort(int a[], int low, int high)
{
    if(low >= high)
    {
        return;
    }
    int first = low;
    int last = high;
    int key = a[first];/*用字表的第一个记录作为枢轴*/
 
    while(first < last)
    {
        while(first < last && a[last] >= key)
        {
            --last;
        }
 
        a[first] = a[last];/*将比第一个小的移到低端*/
 
        while(first < last && a[first] <= key)
        {
            ++first;
        }
         
        a[last] = a[first];    
/*将比第一个大的移到高端*/
    }
    a[first] = key;/*枢轴记录到位*/
    Qsort(a, low, first-1);
    Qsort(a, first+1, high);
}
int main()
{
    int a[] = {57, 68, 59, 52, 72, 28, 96, 33, 24};
 
    Qsort(a, 0, sizeof(a) / sizeof(a[0]) - 1);/*这里原文第三个参数要减1否则内存越界*/
 
    for(int i = 0; i < sizeof(a) / sizeof(a[0]); i++)
    {
        cout << a[i] << "";
    }
     
    return 0;
}/*参考数据结构p274(清华大学出版社，严蔚敏)*/
```

---