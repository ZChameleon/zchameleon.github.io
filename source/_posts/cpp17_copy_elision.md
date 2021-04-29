---
title: 'C++17 Guaranteed Copy Elision and Temporary Materialization'
date: 2021-04-26 14:55:51
tags: 'C++ Core Language'
---


### **Introduction**
---

考虑以下代码在 **C++11** 标准下运行会得到何种输出：
```cpp
struct foo {
    foo() { cout << "ctor "; }
    foo(foo const&) { cout << "copy "; }
    foo(foo&&) { cout << "move "; }
    ~foo() { cout << "dtor "; }
};

int main() {
    foo f = foo{};
}
```

现代编译器至少提供两种可能的输出：**ctor move dtor dtor** 和 **ctor dtor**，后者说明编译器应用了某种优化手段（gcc 和 clang 可以添加 -fno-elide-constructors 编译参数阻止这种优化），本文的后续内容中我们将暂时忽略它，仅讨论未进行优化的场景。观察第一种输出，稍加思索就会意识到事实有些反直觉：仅定义一个类型为 foo 的对象，为什么会引发两次 foo 的构造函数调用（一次默认，一次移动）？

从语义上来讲，**foo f = foo{}** 是一个声明（定义）同时也是一个初始化，而 f 的初始化器为 **= foo{}** ，这实际上满足了复制初始化语义，其行为之一便是：当目标对象（被初始化对象）与源对象（等号右侧的对象）为相同类类型时，将以源对象作为实参，经重载决议后调用适当的复制/移动构造函数，完成对目标对象的初始化工作。因此移动构造函数的调用是为了构造 f，但默认构造函数的调用又是怎么回事？

此问题暂且按下不表，目前可以明确的是，复制初始化语义表明须存在一个源对象作为被调用复制/移动构造函数的参数，对于 **foo f = f1** 源对象显而易见是 f1。但对于 **foo f = foo{}** 而言，这个源对象在哪里？


## **Temporary objects**
---

此时应该考虑（等号右侧的）**foo{}** ，根据 C++11 标准 § 5.2.3 描述：
> A simple-type-specifier or typename-specifier followed by a braced-init-list creates a **temporary object** of the specified type direct-list-initialized with the specified braced-init-list, and its value is that temporary object as a **prvalue**.

表达式 **foo{}** 被归类为函数式转型，转型过程中将创建一个被**直接列表初始化**的临时对象，该临时对象便是我们所寻找的应用于复制初始化语义中的源对象。

除了函数式转型表达式，还有很多特定场景将导致临时对象被创建，C++11 标准对此有明确规定（§ 12.2）：
> Temporaries of class type are created in various contexts: binding a reference to a prvalue, returning a prvalue, a conversion that creates a prvalue, throwing an exception, entering a handler, and in some initializations. Even when the creation of the temporary object is avoided, all the semantic restrictions shall be respected as if the temporary object had been created. [ Note: Even if the copy/move constructor is not called, all the semantic restrictions, such as accessibility, shall be satisfied. — end note ]

下面通过一段代码进行举例说明：
```cpp
struct foo {
    foo(int) {}
    foo() { std::cout << "ctor\n"; }
    foo(foo const&) { std::cout << "copy\n"; }
    foo(foo&&) { std::cout << "move\n"; }
    ~foo() { std::cout << "dtor\n"; }
};

foo func() {
    return 42;   // (a) return 语句创建 42 为参数调用转换构造函数所构造的临时对象
}

int main() {
    double&& rd = 2;        // (b) 引用绑定到值为 2.0 的临时对象
    static_cast<double>(2); // (c) 转换创建值为 2.0 的临时对象
    foo f = 42;             // (d) 初始化创建以 42 为参数调用转换构造函数所构造的临时对象
}
```

根据上述内容，我们可以发现产生临时对象的上下文中似乎都有着 prvalue 的直接参与，而且标准对 prvalue 的描述也完全涉指了临时对象（C++11 § 3.10）：
> * An **lvalue** designates a function or an object.
> * An **xvalue** (an “eXpiring” value) also refers to an object, usually near the end of its lifetime. An xvalue is the result of certain kinds of expressions involving rvalue references.
> * A **glvalue** (“generalized” lvalue) is an lvalue or an xvalue.
> * An **rvalue** is an xvalue, a temporary object or subobject thereof, or a value that is not associated with an object.
> *  A **prvalue** (“pure” rvalue) is an rvalue that is not an xvalue.

这样看来似乎存在于 C++11 标准定义下的临时对象和 prvalue 之间的关系是暧昧不清的，甚至可以说在大部分场景中 prvalue 都可以被直接当作临时对象看待。


## **Copy Elision**
---

此时再回到一开始的问题，答案便显而易见了，foo 的默认构造函数正是由于临时对象的创建而被调用的，而稍微认真思考一下就会意识到，这个临时对象从被创建到被销毁，唯一的作用就是给 f 提供了初始值，从结果上讲这和直接构造 f 是没有区别的，那为何还要多此一举？

幸运的是为避免这类额外开销，C++11 标准提供了一种实现可选的优化方案（§ 12.8）：
> When **certain criteria** are met, an implementation is allowed to omit the copy/move construction of a class object, even if the copy/move constructor and/or destructor for the object have **side effects**. In such cases, the implementation treats the source and target of the omitted copy/move operation as simply two different ways of referring to the same object, and the destruction of that object occurs at the later of the times when the two objects would have been destroyed without the optimization. This elision of copy/move operations, called **copy elision** ...

当满足某些**特定前提**时，即使复制/移动构造函数和析构函数拥有**副作用**，也允许实现省略该复制/移动过程。在这种情况下，实现将被省略的复制/移动操作的源和目标简单视作指代同一对象的两种不同方式，并且该对象的析构发生于如同不进行优化的情况下两个对象应被析构的时刻的较迟者。该行为被称为 **Copy Elision（复制消除）**。

至少有两种典型场景会发生复制消除：
* 以相同类类型的临时对象初始化目标对象时，允许将该临时对象直接构造于目标对象中。而当临时对象作为 return 语句的操作数时，此变种被称为 **RVO - Return Value Optimization**。
* 当函数的 return 语句中表达式指代一个和该函数返回类型为相同类类型的自动对象（不包括函数参数）时，允许将该自动对象直接构造于返回值中，此变种被称为 **NRVO - Named Return Value Optimization**。

现代编译器都会默认启用该项优化（如不指定 -fno-elide-constructors），并尽可能在一切规范允许的场景下应用该优化。直接于目标对象中构造，即省去了临时对象的创建，同时也省去了复制/移动过程，其所带来程序性能上的提升是显而易见的，尤其是对于尺寸较大的对象（大尺寸对象的移动构造应用逐字节复制成本依旧很高）。但同时我们也不能忽略一个可能影响程序的**正确性**问题，那就是复制/移动构造函数和析构函数的 **Side Effects（副作用）**。

通常来讲，副作用是指在求值一个表达式的过程中所引发的一系列除值计算以外的操作。在 C++ 语言中，其包括但不限于：访问 volatile 泛左值所指代的对象，修改任意对象，调用库 I/O 函数，或调用任何包含这些操作的函数。在我们的例子中，初始化 f 所产生的副作用就是向输出流写入字符串 **ctor** 和 **move**，而在应用了复制消除后，初始化 f 所产生的副作用就只有输出字符串 **ctor** 了（省略了移动过程）。

那么为什么说复制消除可能影响程序的正确性呢？对我们的例子而言，在应用优化后，程序的行为显然已经发生了改变（终端输出的内容被改变），不过这部分逻辑是我们有意为之，目的就是为了观察一系列有关于对象初始化的行为，所以并不算影响程序正确性。但假使有这样一个程序，它的一些关键行为依赖某个类的复制/移动构造函数或析构函数所产生的副作用，那应用复制消除就可能带来无法预料的后果。

不需太过担心的是，委员会既通过了这项优化提案，其决定必然是经过深思熟虑的，主要理由是：实践中通常不会在复制/移动构造函数和析构函数中涉及无关对象构造/析构的逻辑，更少可能会直接影响到某种全局状态，因此该优化实际上对程序的整体行为影响较小或完全无影响，又对比其所带来的可观性能提升，缺陷完全可以接受。

同时该讨论也为我们提供了一项 Coding Guideline（编码指导）：**若非必要，切忌依赖复制/移动构造函数或析构函数的副作用**。
