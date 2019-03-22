# Python 常用的内置算法与数据结构

## 常用的内置算法与数据结构
数据结构/算法|语言内置|内置库
-----------|------|----
线性结构|list(列表)/tuple(元组)|array(数组，不常用)/collections.namedtuple
链式结构| |collections.deque(双端队列)
字典结构|dict(字典)|collections.Counter(计数器)/collections.OrderedDict(有序字典)
集合结构|set(集合)/frozenset(不可变集合)| 
排序算法|sorted| 
二分算法| |bisect模块
堆算法| |heapq模块
缓存算法| |functions.lru_cache(Least Recent Used, Python3)

## Python dict 底层结构
dict 底层使用哈希表
- 为了支持快速快速查找使用了哈希表作为底层结构
- 哈希表平均查找时间复杂度为 O(1)
- CPython 解释器使用二次探查解决哈希冲突问题

### 如何解决哈希冲突和扩容
#### 哈希冲突
##### 1. 链接法
##### 2. 探查法
- 线性探查
- 二次探查

#### 扩容

## Python list/tuple 区别
1. 都是线性结构，支持下标访问
2. tuple 保存的引用不可变, 你不能替换掉被引用的这个对象，但是如果被引用的对象本身是一个可变对象，则可以修改这个引用指向的可变对象的内容
3. list 不能作为字典的 key, tuple 可以(可变对象不可 hash)

## 手动实现一个 LRUCache
```python
from collections import OrderedDict

class LRUCache:
    
    def __init__(self, capacity=128):
        self.od = OrderedDict()
        self.capacity = capacity
        
    def get(self, key):
        if key in self.od:
            val = self.od[key]
            self.od.move_to_end(key)
            return val
        else:
            return -1
    
    def put(self, key, value):
        if key in self.od:
            del self.od[key]
            self.od[key] = value
        else:
            self.od[key] = value
            if len(self.od) > self.capacity:
                self.od.popitem(last=False)

```