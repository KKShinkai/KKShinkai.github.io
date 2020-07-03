---
title: "Rust 的背景"
date: "July 3, 2020"
author: "Kk Shinkai"
---

> 🚧 还没有写完, 计划要写 (1) **检查悬垂指针**, (2) **RAII, 所有权和智能指针**, (3) **Trait 与类型系统** 和 (4) **函数式程序的基础设施** 这几部分. 目前进度 (1/4)

很多软件系统, 包括操作系统, 驱动程序, 文件服务器和数据库, 有需要细粒度的 (fine-grained) 控制数据的表示和资源管理. 类似系统最常用的语言是 C, 但是在进行低级 (low-level) 操作时, C 默许了各式各样的危险, 如强制类型转换, 缓冲区溢位, 空指针错误和内存泄漏. 更高级的 (high-level) 的语言能避免这些弊端, 但与此同时, 它们也往往无法为程序员提供对低级系统的控制权.

实际上, 我们希望有这样一门语言, 在保持对资源细粒度的控制时, 具备如下性质:

-   类型系统是可靠 (sound) 的, 静态 (static) 的, 在编译期时候能够 (尽可能的) 发现型别错误, 以增进最终程序的可靠性. 并且类型系统应该尽可能的强大, 以解决静态类型语言的灵活性问题.
-   程序永远不会解引用 (dereference) 任何悬垂指针 (dangling pointer), 解引用悬垂指针的错误会在编译期 (compile-time) 被提出, 运行时 (run-time) 既不会也不需要进行检查.
-   语言提供操控对象布局, 放置位置和生命周期 (lifetime) 的机制. 编译器能协助程序员管理和释放内存, 可能存在内存问题时, 编译器能够及时发现并报告错误.
-   语言本身是简单易用的, 拥有友好的错误提示, 完备的文档, 包管理器和构建工具, 良好的 (带类型推断的) 集成开发环境等等.

总结起来便是 Rust 官网上的: Performance (高性能), Reliability (可靠性) 和 Productivity (生产力).

## 检查悬垂指针

在 C 中, 局部 (local) 变量总是在栈 (stack) 上分配的, 进入和离开作用域也分别代表了变量的分配和回收, 作用域结束后会, 该变量就不再存在. 在下例中, 

```c
point_t *run() {
    point_t p = {.x = 1, .y = 2}; // <-- 'p' is valid
    do_something_with(&p);
    return &p; // Address of stack memory associated with local variable
               // 'p' returned
} // <-- 'p' is no longer valid
```

使用 `malloc` 函数在堆 (heap) 上动态申请的内存则不受作用域影响, 直到被 `free` 函数释放 (或程序退出后由操作系统回收) 为止都会一直存在.

```c
point_t *run() {
    point_t *p = (point_t *)malloc(sizeof(point_t)); // <-- 'p' is valid.
    p->x = 1, p->y = 2;
    do_something_with(p);
    return p;
} // <-- point is still valid
```

手工内存管理 (manual memory management) 非常简单易懂, 但同时也意味着程序员必须记住每个变量的生命周期, 使用之后手动释放, 并确认释放之后不再使用. 由于程序往往是很复杂的, 因此这种策略不可避免的导致了不少无效内访问和内存泄漏的问题.

Cyclone 是一个 C 的 “安全 (safe)” 的方言, 在 Cyclone 里, 前一个例子中的 C 代码会被类型检查器 (type-checker) 拒绝. 编译器会追踪指针 `p` 和它指向的结构体对象的生命周期, 并发现在函数返回后, 结构体对象已经被回收, 但 `p` 仍然存活, 返回值将会成为一个悬垂指针. 这样的代码是不被允许的.

Cyclone 能做到这些, 源于它使用的基于区域的 (region-based) 内存管理策略. Cyclone 会给每个对象分配一个与其生命周期相符的区域 (region) 标识符, 每个指针类型也包含一个区域标识符, 当指针要指向一个对象时, Cyclone 会检查二者的区域是否是相容的.

```cpp
int f() {
    int x = 0;
    int *@region(`f) y = &x; // <-- `f 为函数 f 的区域标识符
    L : {
        int a = 0;
        y = &a; // <-- &a 的类型为 int *@region(`L), 由编译器推断
    }
    return *y;
}
```

这段代码会被 Cyclone 的编译器拒绝, 因为对象 `a` 生存于区域 `` `L `` 中, 这个区域比 `y` 所在的区域 `` `f `` 要小, 如果允许 `y` 指向 `a`, 那么对 `y` 解引用就可能会造成悬垂指针错误. 顺便一提, Cyclone 中的区域标识符是指针类型的一部分, 编译器自动推断, 无需像上文的例子中那样显式注明.

区域系统 (region system) 还有另外一个重要的设施: LIFO 区域. 它支持和栈类似的 LIFO (last-in-first-out) 生命周期系统, 变量的生命和词法作用域相吻合, 但同时还允许动态分配内存.

```cpp
{   region<`r> h;
    ...
}
```

上例的语法声明了一个区域 `` `r ``, 并定义了一个区域句柄 (region handle) `h`, `rmalloc` 和 `rnew` 可以使用该句柄动态的分配内存.

```cpp
int k(int n) {
    int result;
    {   region<`r> h;
        int ?arr = rnew(h) { for i < n : i };
        result = process(h, arr);
    }
    return result;
}
```

当 LIFO 区域 `` `r `` 结束时, 使用句柄 `h` 分配的内存也随之被回收, 这样便无需手动释放, 也不会造成内存泄漏. 事实上, Cyclone 比 C 多了一类内存区域, Cyclone 的内存区域有三种 :

-   栈区域 (stack region) : 局部定义变量的生命周期和词法作用域相吻合, 与 C 语言完全相同.
-   堆区域 (heap region) : 永久区域 `` `H ``, 只能使用 `malloc` 或 `new` 在其上分配空间, 但不能释放, 没有 C 语言中 `free` 函数的类似物.
-   动态区域 (dynamic region) : 临时区域, 使用 `rmalloc` 或 `rnew` 调用区域句柄来分配空间, 无需手动释放, 区域结束后内存自动回收.

到此为止, 我们已经大略的了解了 Cyclone 的指针追踪机制, 实际上, 它很像一种协变的泛型 (长周期被视为短周期的子类型) , 借助类型推断和静态检查, Cyclone 能够在编译期发现并预防悬垂指针的错误. 在 Rust 中我们也能看见 Cyclone 的影子, 在如下的例子中 :

```rust
struct RefPoint {
    x: &i32,
    y: &i32,
}

fn main() {
    let (a, b) = (&1, &2);
    let p = RefPoint { x: a, y: b };

    println!("({}, {})", p.x, p.y);
}
```

编译器会提醒你 :

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:8
  |
2 |     x: &i32,
  |        ^ expected named lifetime parameter
```

因为引用 `x` 和 `y` 指向了 `main` 函数的作用域, 在声明结构体 `let p = RefPoint { x, y }` 时, 必须要确保 `a` 和 `b` 比 `p` 活的更久, 否则一旦 `a` 和 `b` 提前被释放, `x` 和 `y` 就变成了悬垂指针.

为其增加生命周期标记, 编译器就会自动帮我们推断和对比变量的存活时间了.

```rust
struct RefPoint<'a> {
    x: &'a i32,
    y: &'a i32,
}
```

另外, 独占型 (unique) 智能指针和引用计数型 (reference-counted) 智能指针在 Cyclone 中也有相当广泛的应用, 但由于几乎没有人熟悉这门语言, 因此我会在后文中不再以它为例.

(未完)

<!-- ## RAII

RAII 是 C++ 的设计者 Bjarne Stroustrup 提出的概念, 全称是 Resource Acquisition is Initialization, 译为资源获取即初始化. 资源的使用一般分为三个过程: 获取, 使用和销毁. -->

## 参考资料

-   [Cyclone: A safe dialect of C](http://www.cs.umd.edu/projects/cyclone/papers/cyclone-safety.pdf)
-   [Cyclone: A Type-Safe Dialect of C](http://www.cs.umd.edu/~mwh/papers/cyclone-cuj.pdf)
-   [Experience With Safe Manual Memory-Management in Cyclone](http://www.cs.umd.edu/~mwh/papers/ismm.pdf)
-   [Introduction to Regions - Cyclone](http://cyclone.thelanguage.org/wiki/Introduction%20to%20Regions/)
-   [Memory management - Wikipedia](https://en.wikipedia.org/wiki/Memory_management#Dynamic_memory_allocation)
-   [RAII - cppreference.com](https://en.cppreference.com/w/cpp/language/raii)
-   [Region-based memory management - Wikipedia](https://en.wikipedia.org/wiki/Region-based_memory_management)
-   [Region-Based Memory Management in Cyclone](http://www.cs.utah.edu/~regehr/reading/open_papers/cyclone_regions.pdf)
-   [Safe Manual Memory Management in Cyclone](http://www.cs.umd.edu/projects/PL/cyclone/scp.pdf)
-   [Safe Systems Programming in Rust: The Promise and the Challenge](https://people.mpi-sws.org/~dreyer/papers/safe-sysprog-rust/paper.pdf)
-   [The Rust Programming Language](https://doc.rust-lang.org/book/)