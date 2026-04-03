---
confidence: 0.78
created: '2026-04-04'
domain: cpp
path: cpp/smart-pointers
related: []
status: draft
tags:
- cpp
- example
- exclusive-ownership
- memory-management
- smart-pointer
- unique-ptr
- widget
- zero-overhead
title: unique_ptr 介绍与示例
updated: '2026-04-04'
---

# unique_ptr 介绍与示例

# std::unique_ptr<T>

## 简介

`std::unique_ptr` 是**独占所有权**的智能指针。同一时间只有一个 unique_ptr 可以指向某个对象，当 unique_ptr 被销毁时，它所管理的对象也被自动删除。

## 特性

- **独占语义**：不可拷贝（deleted copy constructor/assignment），只能 `std::move`
- **零开销**：编译后与裸指针 +析构函数 delete 等价，无额外内存/性能开销
- **轻量级**：通常就是一个裸指针的大小

## 基本用法

```cpp
#include <memory>
#include <iostream>

// 创建（C++14 起）
auto p1 = std::make_unique<int>(42);        // 单对象
auto p2 = std::make_unique<int[]>(10);       // 数组（C++20 起支持 make_unique<T[]>）

// 访问
std::cout << *p1 << std::endl;    // 解引用：42
std::cout << p1.get() << std::endl; // 获取原始指针

// 所有权转移
auto p3 = std::move(p1);   // p1 变为 nullptr，p3 接管对象
// 此时 p1 不再有效！

// 重置
p3.reset();               // 手动释放，p3 变为 nullptr
p3 = nullptr;              // 等价于 reset()

// 释放 ownership，返回裸指针（需自行负责 delete！）
int* raw = p3.release();
```

## 自定义删除器

```cpp
// 管理 C 风格资源
auto file_closer = [](FILE* f) { if (f) fclose(f); };
using UniqueFile = std::unique_ptr<FILE, decltype(file_closer)>;

UniqueFile file(fopen("data.txt", "r"), file_closer);
// 离开作用域时自动 fclose

// 更通用的写法（C++23 起可用 std::unique_ptr 的推导指南简化）
```

## 数组版本

```cpp
// C++20: make_unique 支持数组
auto arr = std::make_unique<int[]>(10);
arr[0] = 1;           // operator[] 支持
// 离开作用域时自动 delete[]
```

## 常见模式

```cpp
// PIMPL (Pointer to Implementation) 惯用法
class Widget {
public:
    Widget();
    ~Widget();
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// 工厂函数返回 unique_ptr
class Base { public: virtual ~Base() = default; };
class Derived : public Base {};

std::unique_ptr<Base> create() {
    return std::make_unique<Derived>();  // 向上转型 OK
}

// 容器存储多态对象
std::vector<std::unique_ptr<Base>> objects;
objects.push_back(std::make_unique<Derived>());
```

## 注意事项

1. **不要对同一裸指针创建两个 unique_ptr** → 双重 free
2. **move 后原指针为 nullptr**，不要再使用它
3. **`get()` 返回的裸指针不转移所有权**，unique_ptr 仍会 delete 它
4. **unique_ptr 大小取决于删除器类型**——无状态删除器不增加大小
