# Collections

## Counter

Counter是一个计数器, 可以对可迭代对象统计出现的频次。节省了手动进行统计的coding时间。

```py
from collections import Counter

words = ['a', 'b', 'a', 'c', 'b', 'a']
cnt = Counter(words)
print(cnt)          # Counter({'a': 3, 'b': 2, 'c': 1})
print(cnt.most_common(2))  # [('a', 3), ('b', 2)] 取前2高频

# 甚至可以直接统计字符串
print(Counter('abracadabra'))  # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
```

通过`items()`方法获取其中的键值对，如：
```py
cnt = Counter(words)
for k, v in cnt.items():
    ...
```
此处的k为每个item的键，v则为值。

也可以通过`.keys()`或者`.values()`分别获取键或者值。

## defaultdict

通过`defaultdict`可以实现一些直接创建的dict无法达到的效果，比如说直接创建的字典在尝试获取一个不存在的key对应的value时会直接报错。而defaultdict则会自动生成默认值，避免出现报错的情况。

```py
dd = defaultdict(int)
dd['new_key'] += 1
print(dd)# defaultdict(<class 'int'>, {'new_key': 1})
```

## deque

在list中，其实也可以通过`del list[x]`或者`list.pop()`的方法取出元素，但在频繁增删头部元素时，其效率不如`deque`，在`deque`中，可以使用`.appendleft()`和`.popleft()`实现头部的增删。

对于list而言，则只能使用`.pop(i)`和`.insert(i, item)`来插入非末尾的内容了。


## OrderDict

记录了字典的插入顺序，但是`python 3.7`中普通dict已经默认有序，当然，还有`move_to_end()`和`popitem()`控制元素顺序，提高灵活性。