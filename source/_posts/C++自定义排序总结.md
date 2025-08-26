---
title: C++自定义排序总结
categories:
  - 学习记录
  - C++
tags:
  - C++
poster:
  topic: 标题上方的小字
  headline: 大标题
  caption: 标题下方的小字
  color: 标题颜色
date: 2025-08-01 11:14:48
updated: 2025-08-01 11:14:59
description:
cover:
banner:
sticky:
mermaid:
katex:
mathjax:
topic:
author:
references:
comments:
indexing:
breadcrumb:
leftbar:
rightbar:
h1:
type:
---

# C++中的sort与自定义排序

## 基本使用与原理

std::sort 是一个模板函数，常见签名如下：

```cpp
template<class RandomIt>
void sort(RandomIt first, RandomIt last);

template<class RandomIt, class Compare>
void sort(RandomIt first, RandomIt last, Compare comp);
```

- first**,** last：指定排序范围的随机访问迭代器。
- comp：可选的比较器，定义元素顺序，默认为 std::less<T>（基于 operator< 的升序排序）。
- **时间复杂度**：平均和最坏情况均为 O(n log n)。
- **空间复杂度**：通常为 O(log n)，用于递归栈或临时缓冲区。
- **要求**：比较器必须满足严格弱序（strict weak ordering），否则可能导致未定义行为。

### 核心算法：内省排序（Introsort）

C++ 标准未强制指定 std::sort 的实现算法，但大多数现代标准库（如 libstdc++、libc++）采用 **内省排序（Introsort）**。Introsort 由 David Musser 于 1997 年提出，是一种混合排序算法，结合了快速排序（Quicksort）、堆排序（Heapsort）和插入排序（Insertion Sort）的优点，以兼顾效率和鲁棒性。

#### 1. 快速排序

快速排序是 Introsort 的主要算法，步骤如下：

- **选择基准（pivot）**：通常使用三数取中（median-of-three，选取首、尾、中间元素的中位数）或随机选择。
- **分区（partition）**：将元素分为小于等于基准和大于基准的两部分。
- **递归**：对两个子区间递归排序。
- **特点**：
  - 平均时间复杂度：O(n log n)。
  - 最坏情况（如已排序或逆序数组）：O(n²)。
  - 优化：三数取中或随机化基准减少最坏情况概率。
- **实现细节**：
  - 分区方案：常用 Lomuto 或 Hoare 分区算法。
  - 基准选择：三数取中避免极端情况（如已排序输入）。

#### 2. 堆排序

当快速排序的递归深度超过阈值（通常为 2 * log n），Introsort 切换到堆排序，以避免快速排序的最坏情况。

- **步骤**：
  - 构建最大堆（或最小堆，取决于排序顺序）。
  - 反复将堆顶元素移到末尾并调整堆。
- **特点**：
  - 时间复杂度：始终为 O(n log n)。
  - 空间复杂度：O(1)（原地排序）。
  - 适合处理快速排序退化的场景。
- **实现细节**：
  - 使用数组表示堆，通过下标计算父子节点关系。
  - 堆调整（sift-down）确保堆性质。

#### 3. 插入排序

当子区间大小较小时（通常 < 16 或 32 个元素，具体阈值依实现而定），Introsort 切换到插入排序。

- **步骤**：
  - 逐个将元素插入到已排序的子序列中。
- **特点**：
  - 时间复杂度：O(n²)，但在小数组上常数开销低。
  - 适合部分有序或小规模数据。
- **实现细节**：
  - 通常通过循环实现，避免递归。
  - 常用于优化小规模子区间的排序。

#### 原因：

- **快速排序**：平均性能优异，但在最坏情况下（如已排序或重复元素）退化为 O(n²)。
- **堆排序**：保证最坏情况 O(n log n)，但平均性能稍逊，且堆调整开销较高。
- **插入排序**：在小规模数据上高效，减少递归开销。 Introsort 动态切换算法，确保：
- **高效性**：利用快速排序的平均性能。
- **鲁棒性**：通过堆排序避免最坏情况。
- **优化性**：插入排序处理小数组。

## 自定义排序

### 1. 函数指针

定义一个独立的比较函数，签名需为`bool(T, T)`：

```cpp
bool compare(int a, int b) {
    return a > b; // 降序排序
}

std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), compare); // 结果为 {9, 5, 5, 2, 1}
```

### 2. 静态成员函数

在类中定义静态比较函数，避免依赖对象实例：

```cpp
class Sorter {
public:
    static bool compare(int a, int b) {
        return a > b; // 降序排序
    }
};

std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), Sorter::compare);
```

**注意**：非静态成员函数因隐含`this`指针，签名不匹配`std::sort`的要求，因此无法直接使用。

### 3. 使用函数对象

通过定义一个类并重载`operator()`，可以在比较器中存储状态：

```cpp
class Comparator {
    int threshold;
public:
    Comparator(int t) : threshold(t) {}
    bool operator()(int a, int b) const {
        return a > threshold && a < b; // 仅对大于threshold的元素降序排序
    }
};

std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), Comparator(3));
```

### 4. 使用Lambda表达式（C++11及以上）

Lambda表达式提供了一种简洁的方式定义比较器，并可捕获外部变量：

```cpp
int threshold = 3;
std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), [threshold](int a, int b) {
    return a > threshold && a < b;
});
```

### 5. **使用`std::function`**

C++11引入的`std::function`可以包装任何可调用对象（包括函数指针、Lambda、函数对象等），提供更灵活的方式：

```cpp
#include <functional>
std::function<bool(int, int)> comp = [](int a, int b) {
    return a > b; // 降序排序
};

std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), comp);
```

**适用场景**：当需要在运行时动态选择比较器时，`std::function`非常有用，但会引入少量性能开销。

### 6. **使用标准比较器（如`std::greater`、`std::less`）**

C++标准库提供了预定义的比较器（如`<functional>`中的`std::greater`和`std::less`），可直接用于简单排序需求：

```cpp
#include <functional>
std::vector<int> vec = {5, 2, 9, 1, 5};
std::sort(vec.begin(), vec.end(), std::greater<int>()); // 降序排序
std::sort(vec.begin(), vec.end(), std::less<int>());   // 升序排序
```

**优点**：无需手动定义比较器，代码简洁，适合常见升序或降序需求。

### 7. **重载`operator<`**

对于自定义结构体或类，可以通过重载`operator<`来定义默认排序规则，省去显式比较器：

```cpp
struct Person {
    std::string name;
    int age;
    bool operator<(const Person& other) const {
        return age < other.age; // 按年龄升序
    }
};

std::vector<Person> people = {{"Alice", 25}, {"Bob", 30}, {"Charlie", 20}};
std::sort(people.begin(), people.end()); // 使用operator<，按年龄升序
```

**适用场景**：当类型有自然的排序规则且不需要多种排序方式时，重载`operator<`是最简洁的方案。

### 自定义结构体或类的排序

当排序对象是自定义结构体或类时，可以结合上述方法。例如，使用Lambda：

```cpp
std::vector<Person> people = {{"Alice", 25}, {"Bob", 30}, {"Charlie", 20}};

// 按名字字典序排序
std::sort(people.begin(), people.end(), [](const Person& a, const Person& b) {
    return a.name < b.name;
});
```





