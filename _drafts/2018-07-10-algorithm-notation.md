---
layout: post
title:  "算法复杂度标记法"
date:   2018-07-10 18:00:00 +0800
categories: algorithm
---

### 算法复杂度的推演

```java
//排序算法
public static int[] insertionSort(int[] array) {
    for (int i = 1; i < array.length; i++) {
        int tmp = array[i];
        int j = i - 1;
        while (j >= 0 && array[j] > tmp) {
            array[j + 1] = array[j];
            j--;
        }
        array[j + 1] = tmp;
    }
    return array;
}
```

$\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$

### 大欧表示法

表示上限

### 大欧米伽表示法

表示下限

### 西塔表示法

表示渐进

### 小欧表示法

### w表示法