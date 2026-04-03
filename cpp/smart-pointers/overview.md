---
confidence: 1.0
created: '2026-04-04'
domain: cpp
path: cpp/smart-pointers
related: []
status: draft
tags:
- c++11
- cpp
- memory-management
- overview
- shared-ptr
- smart-pointer
- smart-pointers
- unique-ptr
- weak-ptr
title: 智能指针总览
updated: '2026-04-04'
---

# 智能指针总览

# C++ 智能指针 (Smart Pointers)

## 概览

C++11 引入的智能指针模板类，定义在 `<memory>` 头文件中，用于自动管理堆对象的生命周期，避免内存泄漏。

## 三大智能指针

| 指针 | 所有权模型 | 可拷贝 | 核心场景 |
|---|---|---|---|
| `std::unique_ptr<T>` | 独占所有权 | ❌ 不可拷贝，可移动 | **默认首选**，零开销抽象 |
| `std::shared_ptr<T>` | 共享所有权（引用计数） | ✅ 可拷贝 | 多个所有者持有同一对象 |
| `std::weak_ptr<T>` | 弱引用（不增加计数） | ✅ 可拷贝 | 打破 shared_ptr 循环引用，观察者模式 |

## 决策树

```
需要指针？
├─ 对象可以放在栈上？ → 别用指针，直接值语义
├─ 需要动态分配？
│  ├─ 只有一个所有者？ → unique_ptr （首选）
│  ├─ 多个所有者？     → shared_ptr
│  └─ 可能存在环？      → 其中一端用 weak_ptr
└─ 需要 C API 互操作？  → unique_ptr + 自定义删除器
```

## 核心原则

1. **永远用 `make_unique` / `make_shared`**，不要手动 `new`
2. **默认用 unique_ptr**，只在需要共享时升级到 shared_ptr
3. **避免循环引用** —— 所有关系用 shared_ptr，知道关系用 weak_ptr
4. **不要把同一个裸指针交给多个智能指针管理**
5. **unique_ptr 是 zero-cost abstraction** —— 编译后等价于裸指针 + 析构时 delete
