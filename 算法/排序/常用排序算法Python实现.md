# 选择排序、插入排序、希尔排序、归并排序、快速排序、堆排序Python实现

## 冒泡排序

```python
def bubble_sort(arr):
    if len(arr) <= 1:
        return arr
    for i in range(len(arr) - 1, 0, -1):
        flag = False
        for index in range(i):
            if arr[index] > arr[index + 1]:
                arr[index], arr[index + 1] = arr[index + 1], arr[index]
                flag = True
        if not flag:
            break
```

## 选择排序

``` python
def selection_sort(list):
    length = len(list)
    for i in range(length):
        min = i
        for index in range(i + 1, length):
            if list[index] < list[min]:
                min = index
        list[i], list[min] = list[min], list[i]
```

## 插入排序

```python
def insertion_sort(list):
    length = len(list)
    for i in range(1, length):
        for index in range(i, 0, -1):
            if list[index] < list[index - 1]:
                list[index], list[index - 1] = list[index - 1], list[index]
            print(index, list)

```

## 希尔排序

```python
def shell_sort(list):
    length = len(list)
    h = 1
    while h < length / 3:
        h = int(h * 3) + 1
    while h >= 1:
        for i in range(h, length):
            for index in range(i, h - 1, -h):
                print(i, index, list)
                if list[index] < list[index - h]:
                    list[index], list[index - h] = list[index - h], list[index]
                print(i, index, list)
        h = int(h / 3)

```

## 归并排序

```python
def merge_sort(list):
    if len(list) > 1:
        mid = len(list) // 2
        left_half = list[:mid]
        right_half = list[mid:]

        merge_sort(left_half)
        merge_sort(right_half)
        i = j = k = 0
        while i < len(left_half) and j < len(right_half):
            if left_half[i] < right_half[j]:
                list[k] = left_half[i]
                i = i + 1
            else:
                list[k] = right_half[j]
                j = j + 1
            k = k + 1

        while i < len(left_half):
            list[k] = left_half[i]
            i = i + 1
            k = k + 1

        while j < len(right_half):
            list[k] = right_half[j]
            j = j + 1
            k = k + 1
```

## 快速排序

```python
import random

def quick_sort(data):
    def _quick_sort(data, low, high):
        if low >= high:
            return
        index = partition(data, low, high)
        _quick_sort(data, low, index - 1)
        _quick_sort(data, index + 1, high)

    return _quick_sort(data, 0, len(data) - 1)

def partition(arr, low, high):
    pivot = random.randint(low, high)
    arr[pivot], arr[high] = arr[high], arr[pivot]
    small = low - 1
    for index in range(low, high):
        if arr[index] < arr[high]:
            small += 1
            if small != index:
                arr[index], arr[small] = arr[small], arr[index]
    small += 1
    arr[small], arr[high] = arr[high], arr[small]
    return small
```

## 堆排序

```python
def heap_sort(arr):
    length = len(arr)
    for i in range(length, -1, -1):
        heapify(arr, length, i)
    for i in range(length - 1, 0, -1):
        arr[i], arr[0] = arr[0], arr[i]
        heapify(arr, i, 0)

def heapify(arr, length, i):
    largest = i
    l = 2 * i + 1
    r = 2 * i + 2
    if l < length and arr[i] < arr[l]:
        largest = l
    if r < length and arr[largest] < arr[r]:
        largest = r
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, length, largest)
```

