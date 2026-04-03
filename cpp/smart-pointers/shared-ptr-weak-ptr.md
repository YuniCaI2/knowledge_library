---
confidence: 1.0
created: '2026-04-04'
domain: cpp
path: cpp/smart-pointers
related: []
status: draft
tags:
- circular-reference
- cpp
- example
- node
- reference-counting
- shared-ptr
- smart-pointer
- weak-ptr
title: shared_ptr 与 weak_ptr 介绍与示例
updated: '2026-04-04'
---

# shared_ptr 与 weak_ptr 介绍与示例

# std::shared_ptr<T> 与 std::weak_ptr<T>

## std::shared_ptr<T> — 共享所有权的智能指针

### 简介

`shared_ptr` 通过**引用计数**机制实现共享所有权。多个 shared_ptr 可以指向同一个对象，最后一个 shared_ptr 销毁时对象被自动删除。

### 内部结构

```
┌─────────────┐     ┌──────────────────┐     ┌──────────┐
│ shared_ptr A │────▶│  Control Block   │────▶│   对象 T  │
│ shared_ptr B │────▶│  ┌────────────┐  │     └──────────┘
│ shared_ptr C │────▶│  │ strong_cnt │  │
│                │     │  │ weak_cnt   │  │
│                │     │  │ deleter    │  │
│                │     │  │ allocator  │  │
│                │     │  └────────────┘  │
└─────────────────┘     └──────────────────┘
```

- **控制块 (Control Block)** 存储引用计数、弱引用计数、删除器、分配器
- **strong count == 0 时销毁对象，weak count == 0 时销毁控制块**
- 引用计数的增减是**原子操作**（线程安全但有开销）

### 基本用法

```cpp
#include <memory>
#include <iostream>

// 创建
auto sp1 = std::make_shared<std::string>("Hello");  // 推荐！一次分配
auto sp2 = std::shared_ptr<std::string>(new std::string("Hello")); // 两次分配

// 拷贝（引用计数 +1）
auto sp3 = sp1;          // strong_count: 2
{
    auto sp4 = sp1;      // strong_count: 3
} // sp4 离开作用域，strong_count: 2

// 查看引用计数
std::cout << sp1.use_count() << std::endl;  // 输出: 2

// 检查是否唯一拥有
if (sp1.unique()) { /* ... */ }

// 重置
sp3.reset();             // strong_count: 1
```

### ⚠️ 坑点与最佳实践

```cpp
// ====== 坑 1：同一裸指针给两个 shared_ptr =======
int* raw = new int(42);
auto bad1 = std::shared_ptr<int>(raw);
auto bad2 = std::shared_ptr<int>(raw);  // ❌ 双重 delete！崩溃！

// 解决：始终从第一个 shared_ptr 拷贝
auto good1 = std::shared_ptr<int>(new int(42));
auto good2 = good1;  // ✅ 共享控制块
// 或者直接都用 make_shared

// ====== 坑 2：this 指针问题 =======
struct Node {
    std::shared_ptr<Node> parent;
    std::shared_ptr<Node> child;
    
    void set_child(std::shared_ptr<Node> c) {
        child = c;
        c->parent = /* ??? */;  // 不能传 this！
    }
};

// 正确做法：继承 enable_shared_from_this
struct Node : std::enable_shared_from_this<Node> {
    std::shared_ptr<Node> parent;
    std::shared_ptr<Node> child;
    
    void set_child(std::shared_ptr<Node> c) {
        child = c;
        c->parent = shared_from_this();  // ✅ 返回 managing this 的 shared_ptr
    }
};

// ====== 坑 3：循环引用导致内存泄漏 =======
struct A { std::shared_ptr<B> b; };  // ❌
struct B { std::shared_ptr<A> a; };  // ❌ 循环引用

auto a = std::make_shared<A>();
auto b = std::make_shared<B>();
a->b = b;   // a.ref=2, b.ref=2
b->a = a;   // a.ref=2, b.ref=2  永远不会归零！

// ====== 性能注意 =======
// make_shared vs 直接构造：
// - make_shared: 一次内存分配（对象+控制块连续），但不能提前释放对象内存
// - new + shared_ptr 构造：两次分配，但对象可以和 weak_count 一起释放
```

### 自定义删除器与分配器

```cpp
// 自定义删除器（不像 unique_ptr 影响类型）
auto deleter = [](MyObj* p) { 
    p->cleanup(); 
    delete p; 
};
std::shared_ptr<MyObj> sp(new MyObj(), deleter);
// shared_ptr 的删除器不影响类型！不同删除器的 shared_ptr 可以互相赋值

// 别名构造函数：共享所有权但指向不同对象
struct Base { int x; };
derived : public Base { int y; };

auto derived_sp = std::make_shared<Derived>();
auto base_alias = std::shared_ptr<Base>(derived_sp, derived_sp.get());
// base_alias 和 derived_sp 共享控制块，但 base_alias 指向 Base 子对象
```

---

## std::weak_ptr<T> — 弱引用智能指针

### 简介

`weak_ptr` 是 `shared_ptr` 的观察者 companion。它**不增加强引用计数**，不拥有对象，因此不影响对象的生命周期。用于打破循环引用或实现缓存模式。

### 基本用法

```cpp
#include <memory>
#include <iostream>

std::weak_ptr<int> wp;

{
    auto sp = std::make_shared<int>(42);
    wp = sp;                    // weak_count: 1, strong_count: 1
    
    // weak_ptr 不能直接解引用，必须 lock() 提升为 shared_ptr
    if (auto locked = wp.lock()) {
        std::cout << *locked << std::endl;  // 输出: 42，对象还活着
    }
    
    std::cout << wp.expired() << std::endl;  // false，对象仍存在
} // sp 销毁，strong_count -> 0，对象被删除

std::cout << wp.expired() << std::endl;  // true，对象已死
if (auto locked = wp.lock()) {
    // 不会进入这里，lock() 返回空 shared_ptr
}
```

### 典型应用场景

```cpp
// ====== 场景 1：打破循环引用 =======
// "拥有"关系用 shared_ptr，“知道”关系用 weak_ptr
struct Student {
    std::string name;
};

struct Course {
    std::string name;
    std::vector<std::shared_ptr<Student>> students;  // 课程拥有学生列表
};

struct Student {
    std::string name;
    std::vector<std::weak_ptr<Course>> courses;  // 学生只是"知道"选了哪些课
};                                                   // 避免循环引用

// ====== 场景 2：观察者模式 / 缓存 =======
template<typename Key, typename Value>
class Cache {
public:
    void put(const Key& k, std::shared_ptr<Value> v) {
        cache_[k] = v;  // weak_ptr 不阻止外部释放
    }
    
    std::shared_ptr<Value> get(const Key& k) {
        auto it = cache_.find(k);
        if (it != cache_.end() && !it->second.expired()) {
            return it->second.lock();  // 对象还在就复用
        }
        return nullptr;  // 已过期或不存在
    }
    
private:
    std::unordered_map<Key, std::weak_ptr<Value>> cache_;
};

// ====== 场景 3：对象查找表 =======
// 当你需要一个全局注册表但不希望阻止对象被销毁时
std::unordered_map<int, std::weak_ptr<Resource>> resource_registry;

int register_resource(std::shared_ptr<Resource> res) {
    int id = next_id++;
    resource_registry[id] = res;  // 注册但不延长生命周期
    return id;
}
```

### 关键 API

| 方法 | 说明 |
|---|---|
| `lock()` | 尝试提升为 `shared_ptr`，若对象已销毁则返回空 |
| `expired()` | 检查所观察的对象是否已被销毁 |
| `use_count()` | 返回对应的 `shared_ptr` 引用计数（0 表示已过期） |
| `reset()` | 释放对对象的观察 |

---

## 总结：三者选择指南

| 场景 | 选择 | 原因 |
|---|---|---|
| 函数内部临时对象 | `unique_ptr` | 默认首选，零开销 |
| 所有权在函数间传递 | `unique_ptr` + move | 明确表达所有权转移 |
| 对象需要被多处共享 | `shared_ptr` | 引用计数自动管理生命周期 |
| 可能形成引用环 | 一端改 `weak_ptr` | 避免内存泄漏 |
| 观察者 / 缓存 | `weak_ptr` | 不影响被观察对象生命周期 |
| 包装 C API 资源 | `unique_ptr` + 删除器 | 精确控制清理逻辑 |
