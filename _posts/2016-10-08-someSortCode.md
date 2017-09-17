---
layout: post
title:  "什么是Token"
date:   2016-10-08 18:25:01 +0800
tag: Algorithm
---

### 一些排序算法的时间复杂度
![time complexity]({{'/styles/images/O.jpg'}})

### PopSort
```cpp
void PopSort(int a[],int n)
{
    int temp;
    for(int i=0;i<n-1;i++)
    {
        for(int j=0;j<n-1;j++)
        {
            if (a[j]<a[j+1])
            {
                temp = a[j];
                a[j] = a[j+1];
                a[j+1]=temp;
            }
        }
    }
}
```

### InsertSort
```cpp
void InsertSort(int a[],int n)
{
    int i,j;
    int temp;
    for(i=1;i<n;i++)
    {
        temp = a[i];
        j = i-1;
        for(j=i-1;j>=0&&temp<a[j];j--)
        {
            a[j+1] = a[j];
        }
        a[j+1] = temp;
    }
}
```

### ShellSort
shell排序原理示意图：
![ShellSort]({{'/styles/images/shellSort.jpg'}})
```cpp
void ShellSort(int a[],int len)
{
    int i,j,h;
    int r,temp;
    int x = 0;

    for(r = len/2;r>=1;r/=2)
    {
        for(i=r;i<len;i++)
        {
            temp = a[i];
            j=i-r;
            while(j>=0&&temp<=a[j])
            {
                a[j+r]=a[j];
                j-=r;
            }
            a[j+r]=temp;
        }
    }
}
```
