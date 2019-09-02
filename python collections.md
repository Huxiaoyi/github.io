时间：2019-05

作者：胡家义

# Python高性能容器数据类型collections
Python标准库中的collections模块，是在Python内置数据类型基础上，进行封装提供的几个高效、方便的额外的数据类型：
```python
__all__ = ['deque', 'defaultdict', 'namedtuple', 'UserDict', 'UserList', 'UserString', 'Counter', 'OrderedDict', 'ChainMap']
```
当我们遇到适合使用这些数据结构场景时，如果能合理使用这些数据结构，将起到事半功倍的效果；下面结合实例讲解几个常用的数据类型。

## defaultdict：带有默认值的字典
dict是Python最经常使用的数据结构之一；访问Python原生字典key对应的值(d[key])，当指定的key不存在时，是会抛出KeyError异常的。此类报错常见于各类Python项目。但是，如果使用defaultdict，只要传入一个默认的工厂方法，当访问一个不存在的key时， 会自动生成相应类型的value(defaultdict:dict subclass that calls a factory function to supply missing values)。其中，default factory参数可以指定成list, set, int等各种合法类型。
下面列举几个常见做法做对比：
```python
# coding=utf-8
from collections import defaultdict
str_list = [('J', 1), ('K', 2), ('Q', 3), ('A', 4), ('K', 4), ('A', 1)]

# 1.常见做法：访问字典时，先判断key是否存在
dict1 = {}
for k, v in str_list:
	if k in dict1:
		dict1[k].append(v)
	else:
		dict1[k] = [v]

print(dict1)
运行结果：{'J': [1], 'K': [2, 4], 'Q': [3], 'A': [4, 1]}

# 2.使用dict.setdefault(key, default=None)函数：
dict2 = {}
for k, v in str_list:
	dict2.setdefault(k, []).append(v)
print(dict2)
运行结果：{'J': [1], 'K': [2, 4], 'Q': [3], 'A': [4, 1]}

# 3.使用defaultdict
dict3 = defaultdict(list)

for k, v in str_list:
	dict3[k].append(v)
print(dict3)
运行结果：defaultdict(<class 'list'>, {'J': [1], 'K': [2, 4], 'Q': [3], 'A': [4, 1]})

```

## namedtuple
Python最常用的数据结构之一-tuple，它是不可变的序列，因此有些时候比list更适合做函数返回值，tuple还可以做为字典的key，但是tuple有个明显的缺点，不够灵活。 而namedtuple正好弥补tuple不够灵活的缺点，namedtuple是tuple的子类，通过namedtuple可以非常方便的创建用户自定义类，不再限制于使用索引访问它的元素，可以直接像使用对象属性那样去访问它的元素。
```python
# coding=utf-8

# 创建namedtuple对象方式：typename(类型名)，field_names(类型包含的属性字段)
def namedtuple(typename, field_names, *, verbose=False, rename=False, module=None):
    """Returns a new subclass of tuple with named fields.

	# 创建类模板
    >>> Point = namedtuple('Point', ['x', 'y'])
    >>> Point.__doc__                   # docstring for the new class
    'Point(x, y)'

	# 给类赋值，创建类的实例
    >>> p = Point(11, y=22)             # instantiate with positional args or keywords
    
	# 可以像tuple一样，使用索引访问元素
	>>> p[0] + p[1]                     # indexable like a plain tuple
    33

	# 可以像tuple一样，作为右值用于赋值
    >>> x, y = p                        # unpack like a regular tuple
    >>> x, y
    (11, 22)

	# 可以像普通类对象一样，通过属性字段访问
    >>> p.x + p.y                       # fields also accessible by name
    33

	# 可以方便转化成字典
    >>> d = p._asdict()                 # convert to a dictionary
    >>> d['x']
    11

	# 可以方便地用json字典创建自定义类
    >>> Point(**d)                      # convert from a dictionary
    Point(x=11, y=22)

	# namedtuple不能通过"对象.属性=new value"方式更新属性值，可以调用类方法更新属性值
    >>> p._replace(x=100)               # _replace() is like str.replace() but targets named fields
    Point(x=100, y=22)
    """
```
## deque双端队列(double-ended queue缩写)
Python最常用的数据结构list，也就是C语言中的列表，按索引访问元素很快，但是插入和删除元素很慢；相比原生list，deque除了实现list的append、pop外，它最大优势是可以快速地从队列头插入(appendleft)和删除对象(popleft)，时间复杂度均为O(1)，而list(insert(0, v)、pop(0))则是O(n)。 此外，deque是线程安全的，可以同时从deque集合的左边和右边进行操作而不会有影响，常见用法参见多益云项目写日志模块。

## OrderedDict有序字典
Python原生字段由于hash特性，是无序的，OrderedDict继承自dict，将字典每个元素处理为链表的节点元素，从而保证字典元素的有序性。

```python
# coding=utf-8
from collections import OrderedDict
items = [('A', 1), ('B', 2), ('C', 3), ('D', 4)]

regular_dict = dict(items)
ordered_dict = OrderedDict(items)

print('regular_dict:%s' % regular_dict)
for k, v in regular_dict.items():
	print(k, v)

print('ordered_dict:%s' % ordered_dict)
for k, v in ordered_dict.items():
	print(k, v)

# move_to_end用于将某个元素移到字典开头或(last=False)者末尾(last=True)
ordered_dict.move_to_end('A',last=True)
print('ordered_dict:%s' % ordered_dict)
for k, v in ordered_dict.items():
	print(k, v)
运行结果：
regular_dict:{'A': 1, 'B': 2, 'C': 3, 'D': 4}
A 1
B 2
C 3
D 4
ordered_dict:OrderedDict([('A', 1), ('B', 2), ('C', 3), ('D', 4)])
A 1
B 2
C 3
D 4
ordered_dict:OrderedDict([('B', 2), ('C', 3), ('D', 4), ('A', 1)])
B 2
C 3
D 4
A 1
```

## Counter计数器
作为collections模块最好用的数据结构之一，Counter，继承自dict，最常见的用途是统计字符串中各个字符的出现的频次，以及返回TOP-K。
```python
# coding=utf-8
class Counter(dict):
    '''Dict subclass for counting hashable items.  Sometimes called a bag
    or multiset.  Elements are stored as dictionary keys and their counts
    are stored as dictionary values.

    >>> c = Counter('abcdeabcdabcaba')  # count elements from a string

    >>> c.most_common(3)                # three most common elements
    [('a', 5), ('b', 4), ('c', 3)]

    >>> sorted(c)                       # list all unique elements
    ['a', 'b', 'c', 'd', 'e']
    
    >>> ''.join(sorted(c.elements()))   # list elements with repetitions
    'aaaaabbbbcccdde'
   
    >>> sum(c.values())                 # total of all counts
    15

    >>> c['a']                          # count of letter 'a'
    5

    # 访问不存在的key时，返回0，不会出KeyError异常
    >>> for elem in 'shazam':           # update counts from an iterable
    ...     c[elem] += 1                # by adding 1 to each element's count
    
    >>> c['a']                          # now there are seven 'a'
    7

    >>> del c['b']                      # remove all 'b'
    >>> c['b']                          # now there are zero 'b'
    0
```

拓展：当看到Counter类型时，第一时间就想到用来优化基金捐赠统计和捐赠排行榜问题，基金捐赠功能需要统计每个用户捐赠金额，捐赠排行榜(TOP-8)需要返回捐赠前八用户，前期做法是用dict存储每个用户捐赠金额，然后按字典value统一排序，最后返回有序列表的前八项；显然当数据量庞大时候，全排序效率明显低于TOP-K。
下面模拟一下海量数据排序性能比较：
```python
# coding=utf-8
from collections import Counter

test_list = Counter()
for i in range(30000000):
	test_list[i] = random.random()

# 按字典value全排序
start_time1 = time.time()
order1 = sorted(test_list.items(), key=lambda item: item[1], reverse=True)
print(time.time() - start_time1)

# 使用most_common接口
start_time = time.time()
iLen = len(test_list)
order = test_list.most_common(iLen)
print(time.time() - start_time)

# TOP-8
start_time = time.time()
iLen = len(test_list)
order = test_list.most_common(8)
print(time.time() - start_time)

运行结果：
52.12966990470886
42.76265597343445
3.7619528770446777
```
从上面例子可以明显看出most_common的优势，尤其在TOP-K方面的优势；其中Python的sorted使用的是Timesort算法(Timsort结合了归并排序和插入排序)，它在现实中有很好的效率，而most_common的TOP-K使用的是堆排序，下面为most_common源码：

```python
# coding=utf-8
    # most_common源码
    def most_common(self, n=None):
        '''List the n most common elements and their counts from the most
        common to the least.  If n is None, then list all element counts.

        # Emulate Bag.sortedByCount from Smalltalk
        if n is None:
            return sorted(self.items(), key=_itemgetter(1), reverse=True)
        return _heapq.nlargest(n, self.items(), key=_itemgetter(1))
```