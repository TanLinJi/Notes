```python
# 方式1：用方括号 []（如果键不存在会报错）
info = {'name': '班长', 'id': 100}
print(info['name'])      # 输出: 班长（存在，正常）
print(info['age'])       # 报错！KeyError: 'age'

# 方式2：用 get()（如果键不存在返回 None 或默认值）
print(info.get('name'))  # 输出: 班长
print(info.get('age'))   # 输出: None（不报错）
print(info.get('age', 0)) # 输出: 0（指定默认值）
```



`lambda` 是用来快速创建简单函数的一种语法。

```python
lambda 参数: 返回值表达式

# 普通函数（需要 def、函数名、return）
def add(x, y):
    return x + y

# lambda（一行搞定，没有名字）
lambda x, y: x + y
```

- **核心特点**：没有名字、自动返回、只能写一行
- **常用场景**：配合 `sort()`、`map()`、`filter()` 等函数使用





##### `map()函数`

`map()` 是 Python 内置函数，用于**对可迭代对象中的每个元素应用同一个函数**，返回一个迭代器。

```python
map(函数, 可迭代对象1, 可迭代对象2, ...)
```

| 参数       | 说明                   |
| :--------- | :--------------------- |
| 函数       | 要对每个元素执行的函数 |
| 可迭代对象 | 列表、元组、字符串等   |

```python
lists = [20, 28, 15, 19, 21]
result = list(map(lambda x: x > 20, lists))
print(result)  # [False, True, False, False, True]
```

map 的作用：对列表中的**每一个元素**应用函数，返回**变换后的结果**



##### `filter()函数`

`filter()` 是 Python 内置函数，用于**根据条件筛选可迭代对象中的元素**，返回一个迭代器。

```python
filter(函数, 可迭代对象)
```

| 参数       | 说明                           |
| :--------- | :----------------------------- |
| 函数       | 返回 `True`/`False` 的判断函数 |
| 可迭代对象 | 列表、元组、字符串等           |

```python
def is_even(x):
    return x % 2 == 0

numbers = [1, 2, 3, 4, 5, 6]
result = filter(is_even, numbers)

print(result)        # <filter object at 0x...>（迭代器对象）
print(list(result))  # [2, 4, 6]
```

**filter** 的作用：用函数判断每个元素，**保留**使函数返回 `True` 的元素

`filter()` 对每个元素执行函数：

- 函数返回 `True` → **保留**该元素
- 函数返回 `False` → **丢弃**该元素



##### reduce函数

```python
from functools import reduce
reduce(函数, 可迭代对象, 初始值(可选))
```

`reduce` 的名字：**减少、归约**。把一个序列逐步"缩减"成一个单一的值。(就是求累计和)

```
输入: [1, 2, 3, 4, 5]

逐步累积过程：
  初始: 1
  第1步: 1 + 2 = 3
  第2步: 3 + 3 = 6  
  第3步: 6 + 4 = 10
  第4步: 10 + 5 = 15

输出: 15 (总和)
```

```python
from functools import reduce

# 求和
result = reduce(lambda x, y: x + y, [1, 2, 3, 4, 5])
print(result)  # 15

# 执行过程：
# 第1步: x=1, y=2 → 1+2=3
# 第2步: x=3, y=3 → 3+3=6  
# 第3步: x=6, y=4 → 6+4=10
# 第4步: x=10, y=5 → 10+5=15
```

```python
# 带初始值的reduce
# 有初始值
result = reduce(lambda x, y: x + y, [1, 2, 3], 10)
print(result)  # 16 (10+1+2+3)

# 执行过程：
# 第1步: x=10, y=1 → 11
# 第2步: x=11, y=2 → 13
# 第3步: x=13, y=3 → 16
```

reduce的常见应用

```python
numbers = [3, 8, 1, 9, 4, 6]

# 求最大值
max_val = reduce(lambda x, y: x if x > y else y, numbers)
print(max_val)  # 9

# 求最小值  
min_val = reduce(lambda x, y: x if x < y else y, numbers)
print(min_val)  # 1
```

