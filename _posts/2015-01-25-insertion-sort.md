---
layout: post
title: 插入排序算法
---

###伪代码
```
INSERTION-SORT(A)
	for j = 2 to A.length
		key = A[j]
		i = j - 1
		//把A[j]插入到A[1..j-1]
		while i > 0 and A[i] > key
			A[i + 1] = A[i]
			i = i - 1
		A[i + 1] = key
```

###C实现
```cpp
void insertion_sort(int * arr, int len)
{
	int i, j, key;
	for(j = 1; j < len; j++)
	{
		key = arr[j];
		i = j - 1;
		while(i > 0 && arr[i] > key)
		{
			arr[i + 1] = arr[i];
			i --;
		}
		arr[i + 1] = key;
	}
}
```

###循环不变式
* **初始化**:迭代开始前子数组仅由单个元素组成，明显是排序好的，循环不变式成立
* **保持**:如果迭代前子数组A[i..j-1]是排序好的，A[j]会被插入到合适的位置使下次迭代增加j将保持循环不变式
* **终止**:循环终止条件是j > A.length = n，所以我们有数组A[1..n]已排好序
