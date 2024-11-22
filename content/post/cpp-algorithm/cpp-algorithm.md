---
title: Cpp 算法题语法速查表
description: cheat sheet
date: 2024-11-22
slug: cpp-algorithm
image: 
categories:
    - Cpp
---

> *摔杯不减愁的梅子酒*

## 头文件
```cpp
#include <iostream>
#include <$container>
#include <algorithm>
#include <cmath>
#include <random>
```

## 输入输出
```cpp
std::cin >> num;
std::getline(std::cin, str);
std::cout << num << std::endl;
```

## STL 容器
```cpp
vector
deque
list
stack
queue
priority_queue
set
unordered_set
map
unordered_map
```

## 容器创建方法
```cpp
std::$container<$type...> name;
```

以下侧重形式的一致性，一个容器根据其意义决定是否拥有某个函数  

## （几乎）所有容器都有的函数
```cpp
size()
empty()
clear()
erase()
```

## 长度（元素个数）

### 普通数组
```cpp
sizeof(arr) / sizeof(arr[0]);
```

### STL 容器
```cpp
size()
```

### std::string
作为 STL 容器，可以使用 `size()`  
作为字符串，可以使用 `length()`  

## 容器系列函数
### vector, deque, list
```cpp
push_back(value)
pop_back()
push_front(value)
pop_front()
```

### stack, queue, priority_queue
```cpp
push()
pop()
top()
front()
back()
```

### set, unordered_set, map, unordered_map
```cpp
insert()
find()
```

注意 `find` 函数返回迭代器  

## [] 的使用
vector, deque, string:
```cpp
container[index] = e
```

map, unordered_map:
```cpp
container[key] = value
```

## 迭代器的使用
```cpp
for (auto it = container.begin(); it != container.end(); ++it) {
    // *it / it->first, it->second
}
```

## 算法函数

以 vector 为例：
```cpp
std::sort(vec.begin(), vec.end());
auto it = std::find(vec.begin(), vec.end(), target);
std::copy(vec1.begin(), vec1.end(), vec2.begin());
int count = std::count(vec.begin(), vec.end(), target);
std::reverse(vec.begin(), vec.end());
int sum = std::accumulate(vec.begin(), vec.end(), 0);
auto it = std::lower_bound(vec.begin(), vec.end(), value);
auto it = std::upper_bound(vec.begin(), vec.end(), value);
```
