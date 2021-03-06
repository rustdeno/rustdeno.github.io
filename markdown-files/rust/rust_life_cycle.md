## 1.引言

**所有权**与**生命周期**是  `Rust`  语言非常核心的内容。其实不仅仅是 `Rust` 有这两个概念，在`C/C++` 中也一样是存在的。而几乎所有的内存安全问题也源于对所有权和生命周期的错误使用。只要是不采用垃圾回收来管理内存的程序语言，都会有这个问题。只是 `Rust` 在语言级明确了这两个概念，并提供了相关的语言特性让用户可以显式控制所有权的转移与生命周期的声明。同时编译器会对各种错误使用进行检查，提高了程序的内存安全性。

所有权和生命周期其涉及的语言概念很多，本文主要是对梳理出与“所有权与生命周期”相关的概念，并使用  `UML` 的类图表达概念间的关系，帮助更好的理解和掌握。

**图例说明**

本文附图都是 `UML` 类图，`UML` 类图可以用来表示对概念的分析。表达概念之间的依赖、继承、聚合、组成等关系。图中的每一个矩形框都是一个语义概念，有的是抽象的语言概念，有的是 `Rust` 库中的结构和 `Trait`。

所有图中使用的符号也只有最基础的几个。图 1 对符号体系做简单说明，主要解释一下表达概念之间的关系的符号语言。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d4985fc805b47159de20fced42cd0f4~tplv-k3u1fbpfcp-zoom-1.image)

图 1 UML 符号

**依赖关系：**

依赖是 `UML` 中最基础的关系语义。 以带箭头的虚线表示，`A` 依赖与 `B` 表达如下图。直观理解可以是 `A` “看的见” `B`，而 `B` 可以对 `A` 一无所知。比如在代码中 结构体 `A` 中有 结构体 `B` 的成员变量，或者 `A` 的实现代码中有 `B` 的局部变量。这样如果找不到 `B`，`A` 是无法编译通过的。

**关联关系：**

一条实线连接表示两个类型直接有关联，有箭头表示单向"可见",无箭头表示相互之间可见。关联关系也是一种依赖，但是更具体。有时候两个类型之间的关联关系太复杂，需要用一个类型来表达，叫做关联类型，如例图中的 `H`.

**聚合与组成：**

聚合与组成都是表示的是整体和部分的关系。差别在于“聚合”的整体与部分可以分开，部分可以在多个整体之间共享。而“组成”关系中整体对部分有更强的独占性，部分不能被拆开，部分与整体有相同的生命周期。

**继承与接口实现：**

继承与接口实现都是一种泛化关系，`C` 继承自 `A`，表示 `A` 是更泛化的概念。`UML` 中各种关系语义也可以用 `UML` 自身来表达，如图 2：“关联”和“继承”都是“依赖”的具体体现方式。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbc05fcbfea54cedae4b6aa50ca9e301~tplv-k3u1fbpfcp-zoom-1.image)

图 2 用 UML 表达 UML 自身

**总图**

图 3 是本文的总图，后续各节分局部介绍。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62a6cd02a63249d487cd0b372b487e51~tplv-k3u1fbpfcp-zoom-1.image)

图 3 Rust 所有权与生命周期总图

## 2.所有权与生命周期期望解决的问题

我们从图中间部分开始看起，所谓“所有权”是指对一个变量拥有了一块“内存区域”。这个内存区域，可以在堆上，可以在栈上，也可以在代码段，还有些内存地址是直接用于 `I/O` 地址映射的。这些都是内存区域可能存在的位置。

在高级语言中，这个内存位置要在程序中要能被访问，必然就会与一个或多个变量建立关联关系（低级语言如汇编语言，可以直接访问内存地址）。也就是说，通过这一个或多个变量，就能访问这个内存地址。

这就引出三个问题：

1.  内存的不正确访问引发的内存安全问题
1.  由于多个变量指向同一块内存区域导致的数据一致性问题
1.  由于变量在多个线程中传递，导致的数据竞争的问题

由第一个问题引发的内存安全问题一般有 5 个典型情况：

- 使用未初始化的内存
- 对空指针解引用
- 悬垂指针(使用已经被释放的内存)
- 缓冲区溢出
- 非法释放内存(释放未分配的指针或重复释放指针)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43d24d4277ab43e7b90ecc5bf62cc945~tplv-k3u1fbpfcp-zoom-1.image)

图 4 变量绑定与内存安全的基本概念

这些问题在 `C/C++` 中是需要开发者非常小心的自己处理。 比如我们可以写一段 `C++` 代码，把这五个内存安全错误全部犯一遍。

```cpp
#include <iostream>

struct Point {
	int x;
	int y;
};

Point* newPoint(int x,int y) {
	Point p { .x=x,.y=y };
	return &p; //悬垂指针
}

int main() {
	int values[3]= { 1,2,3 };
	std::cout<<values[0]<<","<<values[3]<<std::endl; //缓冲区溢出

	Point *p1 = (Point*)malloc(sizeof(Point));
	std::cout<<p1->x<<","<<p1->y<<std::endl; //使用未初始化内存

	Point *p2 = newPoint(10,10); //悬垂指针
	delete p2; //非法释放内存

	p1 = NULL;
	std::cout<<p1->x<<std::endl; //对空指针解引用
	return 0;
}

```

这段代码是可以编译通过的，当然，编译器还是会给出警告信息。这段代码也是可以运行的，也会输出信息，直到执行到最后一个错误处“对空指针解引用时”才会发生段错误退出。

`Rust` 的语言特性为上述问题提供了解决方案，如下表所示：

| 问题                             | 解决方案                                                                                                                                                                                                                                                                      |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 使用未初始化的内存               | 编译器禁止变量读取未赋值变量                                                                                                                                                                                                                                                  |
| 对空指针解引用                   | 使用 Option 枚举替代空指针                                                                                                                                                                                                                                                    |
| 悬垂指针                         | 生命周期标识与编译器检查                                                                                                                                                                                                                                                      |
| 缓冲区溢出                       | 编译器检查，拒绝超越缓冲区边界的数据访问                                                                                                                                                                                                                                      |
| 非法释放内存                     | 语言级的 RAII 机制，只有唯一的所有者才有权释放内存                                                                                                                                                                                                                            |
| 多个变量修改同一块内存区域       | 允许多个变量借用所有权，但是同一时间只允许一个可变借用                                                                                                                                                                                                                        |
| 变量在多个线程中传递时的安全问题 | 对基本数据类型用 Sync 和 Send 两个 Trait 标识其线程安全特性，即能否转移所有权或传递可变借用，把这作为基本事实。再利用泛型限定语法和 Trait impl 语法描述出类型线程安全的规则。编译期间使用类似规则引擎的机制，基于基本事实和预定义规则为用户代码中的跨线程数据传递做推理检查。 |

## 3.变量绑定与所有权的赋予

`Rust` 中为什么叫“变量绑定”而不叫“变量赋值"。我们先来看一段 `C++` 代码，以及对应的 `Rust` 代码。

C++:

```cpp
#include <iostream>
 
int main()
{
	int a = 1;
	std::cout << &a << std::endl;   /* 输出 0x62fe1c */
	a = 2;
	std::cout << &a << std::endl;   /* 输出 0x62fe1c */
}

```

Rust:

```rust
fn main() {
	let a = 1;
	println!("a:{}",a);     // 输出1
	println!("&a:{:p}",&a); // 输出0x9cf974
	//a=2;                  // 编译错误，不可变绑定不能修改绑定的值
	let a = 2;              // 重新绑定
	println!("&a:{:p}",&a); // 输出0x9cfa14地址发生了变化
	let mut b = 1;          // 创建可变绑定
	println!("b:{}",b);     // 输出1
	println!("&b:{:p}",&b); // 输出0x9cfa6c
	b = 2;
	println!("b:{}",b);     // 输出2
	println!("&b:{:p}",&b); // 输出0x9cfa6c地址没有变化
	let b = 2;              // 重新绑定新值
	println!("&b:{:p}",&b); // 输出0x9cfba4地址发生了变化
}

```

我们可以看到，在 `C++` 代码中，变量 `a` 先赋值为 1，后赋值为 2，但其地址没有发生变化。`Rust` 代码中，`a` 是一个不可变绑定，执行`a=2`动作被编译器拒绝。但是可以使用 `let` 重新绑定，但这时 `a` 的地址跟之前发生了变化，说明 a 被绑定到了另一个内存地址。`b` 是一个可变绑定，可以使用`b = 2`重新给它指向的内存赋值，`b` 的地址不变。但使用 `let` 重新绑定后，`b` 指向了新的内存区域。

可以看出，"赋值" 是将值写入变量关联的内存区域，"绑定" 是建立变量与内存区域的关联关系，`Rust` 里，还会把这个内存区域的所有权赋予这个变量。

不可变绑定的含义是：将变量绑定到一个内存地址，并赋予所有权，通过该变量只能读取该地址的数据，不能修改该地址的数据。对应的，可变绑定就可以通过变量修改关联内存区域的数据。从语法上看，有 `let` 关键字是绑定, 没有就是赋值。

这里我们能看出 `Rust` 与 `C++` 的一个不同之处。`C++` 里是没有“绑定”概念的。`Rust` 的变量绑定概念是一个很关键的概念，它是所有权的起点。有了明确的绑定才有了所有权的归属，同时解绑定的时机也确定了资源释放的时机。

所有权规则：

- 每一个值都有其所有者变量
- 同一时间所有者变量只能有一个
- 所有者离开作用域，值被丢弃(释放/析构)

作为所有者，它有如下权利：

- 控制资源的释放
- 出借所有权
- 转移所有权

## 4.所有权的转移

所有者的重要权利之一就是“转移所有权”。这引申出三个问题：

1.  为什么要转移？
1.  什么时候转移？
1.  什么方式转移？

相关的语言概念如下图。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c63981e099944376a8261f91062c62b2~tplv-k3u1fbpfcp-zoom-1.image)

图 5 所有权转移

**为什么要转移所有权？**

我们知道，C/C++/Rust 的变量关联了某个内存区域，但变量总会在表达式中进行操作再赋值给另一个变量，或者在函数间传递。实际上期望被传递的是变量绑定的内存区域的内容，如果这块内存区域比较大，复制内存数据到给新的变量就是开销很大的操作。所以需要把所有权转移给新的变量，同时当前变量放弃所有权。所以归根结底，转移所有权还是为了性能。

**所有权转移的时机总结下来有以下两种情况：**

1.  位置表达式出现在值上下文时转移所有权
1.  变量跨作用域传递时转移所有权

第一条规则是一个精确的学术表达，涉及到位置表达式，值表达式，位置上下文，值上下文等语言概念。它的简单理解就是各种各样的赋值行为。能明确指向某一个内存区域位置的表达式是位置表达式，其它的都是值表达式。各种带有赋值语义的操作的左侧是位置上下文，右侧是值上下文。

当位置表达式出现在值上下文时，其程序语义就是要把这边位置表达式所指向的数据赋给新的变量，所有权发生转移。

第二条规则是“变量跨作用域时转移所有权”。

图上列举出了几种常见的跨作用域行为，能涵盖大多数情况，也有简单的示例代码

- 变量被花括号内使用
- match 匹配
- if let 和 While let
- 移动语义函数参数传递
- 闭包捕获移动语义变量
- 变量从函数内部返回

为什么变量跨作用域要转移所有权？

在 `C/C++` 代码中，是否转移所有权是程序员自己隐式或显式指定的。

试想，在 `C/C++` 代码中，函数 `Fun1` 在栈上创建一个 类型 `A` 的实例 `a`， 把它的指针 `&a` 传递给函数 `void fun2(A* param)` 我们不会希望 `fun2` 释放这个内存，因为 `fun1` 返回时，栈上的空间会自动被释放。

如果 `fun1` 在堆上创建 `A` 的实例 `a`， 把它的指针 `&a` 传递给函数 `fun2(A* param)`,那么关于 `a` 的内存空间的释放，`fun1` 和 `fun2` 之间需要有个商量，由谁来释放。`fun1` 可能期望由 `fun2` 来释放，如果由 `fun2` 释放，则 `fun2` 并不能判断这个指针是在堆上还是栈上。归根结底，还是谁拥有 `a` 指向内存区的所有权问题。 `C/C++` 在语言层面上并没有强制约束。`fun2` 函数设计的时候，需要对其被调用的上下文做假定，在文档中对对谁释放这个变量的内存做约定。这样编译器实际上很难对错误的使用方式给出警告。

`Rust` 要求变量在跨越作用域时明确转移所有权，编译器可以很清楚作用域边界内外哪个变量拥有所有权，能对变量的非法使用作出明确无误的检查，增加的代码的安全性。

**所有权转移的方式有两种：**

- 移动语义-执行所有权转移
- 复制语义-不执行转移，只按位复制变量

这里我把 ”复制语义“定义为所有权转移的方式之一，也就是说“不转移”也是一种转移方式。看起来很奇怪。实际上逻辑是一致的，因为触发复制执行的时机跟触发转移的时机是一致的。只是这个数据类型被打上了 `Copy` 标签 `trait`, 在应该执行转移动作的时候，编译器改为执行按位复制。

`Rust` 的标准库中为所有基础类型实现的 `Copy Trait`。

这里要注意，标准库中的

```rust
 impl<T: ?Sized> Copy for &T {}

```

为所有引用类型实现了 `Copy`, 这意味着我们使用引用参数调用某个函数时，引用变量本身是按位复制的。标准库没有为可变借用 `&mut T` 实现“Copy” `Trait` , 因为可变借用只能有一个。后文讲闭包捕获变量的所有权时我们可以看到例子。
