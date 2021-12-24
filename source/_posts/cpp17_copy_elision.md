---
title: 'Copy Elision 的前世今生'
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

现代编译器至少提供两种可能的输出：**ctor move dtor dtor** 和 **ctor dtor**，后者说明编译器应用了某种优化手段（gcc 和 clang 可以添加 -fno-elide-constructors 编译参数阻止这种优化），在后续讨论中我们将暂时忽略它，仅考虑未进行优化的场景。观察第一种输出，稍加思索就会意识到事实有些反直觉：仅定义一个类型为 foo 的对象，为什么会引发两次 foo 的构造函数调用（一次默认，一次移动）？

从语义上来讲，**foo f = foo{}** 是一个声明（定义）同时也是一个初始化，而 f 的初始化器为 **= foo{}** ，这实际上满足了复制初始化语义，其行为之一便是：当目标对象（被初始化对象）与源对象（等号右侧的对象）为相同类类型时，将以源对象作为实参，经重载决议后调用适当的复制/移动构造函数，完成对目标对象的初始化工作。因此移动构造函数便是为了构造 f 而被调用的，但默认构造函数的调用又是怎么回事？

此问题暂且按下不表，思考另一个子问题：复制初始化语义表明须存在一个源对象作为被调用复制/移动构造函数的参数，对于 **foo f = f1** 源对象显而易见是 f1，但对于 **foo f = foo{}** 而言，这个源对象在哪里？


## **Temporary objects**
---

此时考虑（等号右侧的）表达式 **foo{}** ，根据 C++11 标准 § 5.2.3 描述：
> A simple-type-specifier or typename-specifier followed by a braced-init-list creates a **temporary object** of the specified type direct-list-initialized with the specified braced-init-list, and its value is that temporary object as a **prvalue**.

表达式 **foo{}** 被归类为函数式转型，转型过程中将创建一个被**直接列表初始化**的临时对象，该临时对象便是我们所寻找的应用于复制初始化语义中的源对象。

而除了函数式转型表达式，还有很多特定场景将导致临时对象被创建，C++11 标准对此有明确规定（§ 12.2）：
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

因此存在于 C++11 标准定义下的临时对象和 prvalue 之间的关系是暧昧不清的，甚至可以说在大部分场景中 prvalue 都可以被直接当作临时对象看待。


## **Copy Elision**
---

此时再回到一开始的问题，答案便显而易见了，foo 的默认构造函数正是由于临时对象的创建而被调用的，而稍微认真思考一下就会意识到，这个临时对象从被创建到被销毁，唯一的作用就是给 f 提供了初始值，从结果上讲这和直接构造 f 是没有区别的，那为何还要多此一举？

于是为避免此类额外开销，C++11 标准提供了一种实现可选的优化方案（§ 12.8）：
> When **certain criteria** are met, an implementation is allowed to omit the copy/move construction of a class object, even if the copy/move constructor and/or destructor for the object have **side effects**. In such cases, the implementation treats the source and target of the omitted copy/move operation as simply two different ways of referring to the same object, and the destruction of that object occurs at the later of the times when the two objects would have been destroyed without the optimization. This elision of copy/move operations, called **copy elision** ...

当满足某些**特定前提**时，即使复制/移动构造函数和析构函数拥有**副作用**，也允许实现省略该复制/移动过程。在这种情况下，实现将被省略的复制/移动操作的源和目标简单视作指代同一对象的两种不同方式，并且该对象的析构发生于如同不进行优化的情况下两个对象应被析构的时刻的较迟者。该行为被称为 **Copy Elision（复制消除）**。

至少有两种典型场景会发生复制消除：
* 以相同类类型的临时对象初始化目标对象时，允许将该临时对象直接构造于目标对象中。而当临时对象作为 return 语句的操作数时，此变种被称为 **RVO - Return Value Optimization**。
* 当函数的 return 语句中表达式指代一个和该函数返回类型为相同类类型的自动对象（不包括函数参数）时，允许将该自动对象直接构造于返回值中，此变种被称为 **NRVO - Named Return Value Optimization**。

现代编译器都会默认启用该项优化（如不指定 -fno-elide-constructors），并尽可能在一切规范允许的场景下应用该优化。直接于目标对象中构造，即省去了临时对象的创建，同时也省去了复制/移动过程，其所带来程序性能上的提升是显而易见的，尤其是对于尺寸较大的对象（大尺寸对象的移动构造应用逐字节复制成本依旧很高）。但同时我们也不能忽略一个可能影响程序的**正确性**问题，那就是复制/移动构造函数和析构函数的 **Side Effects（副作用）**。

通常来讲，副作用是指在求值一个表达式的过程中所引发的一系列除值计算以外的操作。在 C++ 语言中，其包括但不限于：访问 volatile 泛左值所指代的对象，修改任意对象，调用库 I/O 函数，或调用任何包含这些操作的函数。在我们的例子中，初始化 f 所产生的副作用就是向输出流写入字符串 **ctor** 和 **move**，而在应用了复制消除后，初始化 f 所产生的副作用就只有输出字符串 **ctor** 了（省略了移动过程）。

那么为什么说复制消除可能影响程序的正确性呢？对我们的例子而言，在应用优化后，程序的行为显然已经发生了改变（终端输出的内容被改变），不过这部分逻辑是我们有意为之，目的就是为了观察一系列有关于对象初始化的行为，所以并不算影响程序正确性。但假使有这样一个程序，它的一些关键行为依赖某个类的复制/移动构造函数或析构函数所产生的副作用，那应用复制消除就可能带来无法预料的后果。

不需太过担心的是，委员会既通过了这项优化提案，其决定必然是经过深思熟虑的，主要理由是：实践中通常不会在复制/移动构造函数和析构函数中涉及无关对象构造/析构的逻辑，更少可能会直接影响到某种全局状态，因此该优化实际上对程序的整体行为影响较小或完全无影响，又对比其所带来的可观性能提升，缺陷完全可以接受。

同时该决定实际上明确了一条 Coding Guideline（编码指导）：**若非必要，切忌依赖复制/移动构造函数或析构函数的副作用**。


## **Non-movable objects**
---

那么是否有了 **Copy Elision** 就万事大吉了呢？答案是否定的，考虑如下例子：
```cpp
struct non_movable {
    non_movable() = default;
    non_movable(non_movable&&) = delete;
};

non_movable make() { return {}; }   // ok

auto n1 = make();        // oops!
auto n2 = non_movable{}; // emmm...
```
> **Non Movable** 同时也意味着 **Non Copyable**，因为根据类的特殊成员函数生成规则，如果存在**用户定义或被删除**的移动构造函数，那么**隐式声明**的复制构造函数也将是**被删除**的。

尝试在 C++11/14 标准下编译以上代码会得到类似这样的错误诊断：
```cpp
auto n1 = make();        | error: call to deleted constructor of 'non_movable'
auto n2 = non_movable{}; | error: call to deleted constructor of 'non_movable'
```
> 无论是否应用复制消除结果都是一样的。

这样的结果看似意料之外，但却是情理之中：Copy Elision 仅是一种**可选**的优化手段，考虑其未应用的情况下，对象的初始化工作依旧要通过调用复制/移动构造函数来完成，因此为了语义的一致性，C++11 标准要求无论实际是否应用了该优化，复制/移动构造函数必须**存在且可访问**（§ 12.8）：
> When the criteria for elision of a copy operation are met or would be met save for the fact that the source object is a function parameter, and the object to be copied is designated by an lvalue, overload resolution to select the constructor for the copy is first performed as if the object were designated by an rvalue. If overload resolution fails, or if the type of the first parameter of the selected constructor is not an rvalue reference to the object’s type (possibly cv-qualified), overload resolution is performed again, considering the object as an lvalue. [ Note: This two-stage overload resolution must be performed regardless of whether copy elision will occur. **It determines the constructor to be called if elision is not performed, and the selected constructor must be accessible even if the call is elided.** — end note ]

这样一来我们便无法为不可移动的对象实现工厂函数或应用统一风格的 auto 形式初始化，这也是 C++11 标准下实现的 Copy Elision 的最大缺陷。

> make 函数中的 **return {}** 之所以合法是因为其对应的是直接初始化语义，不需求从临时对象初始化返回值，如果改成 **return non_movable{}** 就无法编译通过，理由同上。？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？retur{} 大概是复制初始化


## **Guaranteed Copy Elision**
---

以上问题随 C++17 标准的发布正式得到解决，其通过一项被称为 **Guaranteed Copy Elision（受保证的复制消除）**的技术，来保证在如前文所述各类除 NRVO 以外的场景中一定不会有临时对象产生，没有临时对象就意味着不必调用复制/移动构造函数，因此也就不需要它们有定义或可访问。

从该项技术的命名来看，其是通过在满足条件的情况下强制应用复制消除来实现的，然而这却是一个具有欺骗性的名字，事实上标准文档中从未提及过 **Guaranteed Copy Elision** 这个名词，实现方式也并非通过强制应用复制消除。

前文曾明确过，凡是出现 prvalue 的场合几乎必有临时对象被创建，也正是这些临时对象带来了不必要的复制/移动开销，所以该项技术选择从根本入手，通过修改 **Value Categories** 的定义，避免了 prvalue 与临时对象有直接关联。根据 C++17 标准 § 6.10 所述：
> * A **glvalue** is an expression whose evaluation determines the identity of an object, bit-field, or function.
> * A **prvalue** is an expression whose evaluation initializes an object or a bit-field, or computes the value of the operand of an operator, as specified by the context in which it appears.
> * An **xvalue** is a glvalue that denotes an object or bit-field whose resources can be reused (usually because it is near the end of its lifetime).
> * An **lvalue** is a glvalue that is not an xvalue.
> * An **rvalue** is a prvalue or an xvalue.

prvalue 是其求值初始化一个对象/位域或计算一个运算符的操作数的值的表达式，进一步的：

> The result of a prvalue is the value that the expression stores into its context. A prvalue whose result is the value V is sometimes said to have or name the value V. The result object of a prvalue is the object initialized by the prvalue; a prvalue that is used to compute the value of an operand of an operator or that has type cv void has no result object.

prvalue 的**结果**是该表达式在其上下文中所存储的值，prvalue 的**结果对象**是被其所初始化的对象，用于计算一个运算符的操作数的值或拥有可能是 cv 限定的 void 类型的 prvalue **没有**结果对象。

同时标准还修改了初始化和返回语句在涉及临时对象场景下的语义，经修改后应用于该场景的直接或复制初始化被定义为：
> If the initializer expression is a prvalue and the cv-unqualified version of the source type is the same class as the class of the destination, the initializer expression is used to initialize the destination object.

以上一节中的代码为例：**make()** 和 **non_movable{}** 都属于 prvalue 表达式，n1 和 n2 则分别为二者的结果对象，整个初始化过程表现为直接以 prvalue 表达式初始化其对应结果对象。并且该性质是**可传递**的，对于 **T x = T{T{T{}}}** ，无论其**看上去**需经历多少次连续初始化，实际行为也始终是直接以最内层表达式 **T{}** 对其结果对象 x 进行单次初始化（这类似一种名为 **Lazy Evaluation** 的 **PL** 领域概念）。

但不可忽略的一点是，在某些（除对象初始化以外的）场景下，临时对象是**有必要**被创建的，例如：
```cpp
struct bar { string str; };

bar&& rrb = {"grimoire"};       // (a) 以 prvalue 表达式初始化引用，需求引用绑定到临时对象
bar{}.str;                      // (b) 在类类型 prvalue 表达式上进行成员访问，需求访问临时对象的成员
type_identity_t<bar[5]>{}[2];   // (c) 在数组类型 prvalue 表达式上进行下标运算，需求访问临时数组的元素
                                // 注：bar[5]{} 不是合法的表达式，需要通过 type alias 创建数组 prvalue
```

为了明确这些场景下的语义，C++17 标准 § 7.4 补充了一项额外规则，称为 **Temporary Materialization（临时量实质化）**：
> A prvalue of type T can be converted to an xvalue of type T. This conversion initializes a temporary object of type T from the prvalue by evaluating the prvalue with the temporary object as its result object, and produces an xvalue denoting the temporary object. T shall be a complete type.

任何 prvalue 可以转换成同类型的 xvalue，此转换以该 prvalue 初始化一个临时对象（将该临时对象作为求值该 prvalue 的结果对象），并产生一个指代该临时对象的 xvalue。满足该转换的场景如下（由 [cppreference](https://en.cppreference.com/w/cpp/language/implicit_conversion#Temporary_materialization) 总结，包含了上述例子中的情况）：
> * when binding a reference to a prvalue.
> * when performing a member access on a class prvalue.
> * when performing an array-to-pointer conversion (see above) or subscripting on an array prvalue.
> * when initializing an object of type std::initializer_list<T> from a braced-init-list.
> * when typeid is applied to a prvalue (this is part of an unevaluated expression).
> * when sizeof is applied to a prvalue (this is part of an unevaluated expression).
> * when a prvalue appears as a discarded-value expression.


## **Summary**
---

至此 C++ 标准通过对与临时对象有关的部分概念进行重新定义，从根本上实现了 **Guaranteed Copy Elision**【仍允许在源表达式为非 prvalue 的情况下应用非强制 **Copy Elision** 作为其补集（目前已有[提案](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2025r1.html)请求强制实施 NRVO）】，并通过新增 **Temporary Materialization** 的概念保证了 Backward Compatibility，因此我们可以在不改动任何原有代码的情况下，通过启用 C++17 标准编译选项成为在性能方面的直接受益者（一如受益于 C++11 标准所引入的移动语义），此外重新定义的值语义体系还有助于编译器实现更完善的 Alias Analysis，可能因此带来的益处将是全面且难以估量的。


## **References**
---
