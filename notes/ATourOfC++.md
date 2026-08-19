

- constexpr
- consteval
- const

# 2 User-Defined Types

## 2.4 Enumerations



## 2.5 Unions

### 为什么使用union？

如果对象不可能同时出现，那么就没有必要存储多个对象，通过当这几个对象放到同一段内存中（最大对象所占的内存长度），从而节省空间。

### 为什么引入variant？

用户必须手动维护一个类型，从而通过类型判断何时使用union内部的那一个类型。variant内部维护了这个类型



# 3 Modularity

## 3.1 Introduction

### 区分接口和实现（实现模块化的第一步）？

- 通过declarations表示接口
- 通过definitions表示实现

declarations制定了用户所需要的所有内容，所以definitions可以在任意其他文件中。

### ODR规则？

只能有一个定义，但声明可以多个。（**One Definition Rule**）

## 3.2 Separate Compilation

### 分离编译如何实现？

- 头文件：将声明放到头文件，然后用户`#include`对应的头文件
- 模块：单独编译模块文件，然后用户import对应的文件（只有export的声明才是用户可见的）

### 3.2.1 Header Files

#### 翻译单元？

每一个独立编译的`.cpp`文件，也就是说有多少个cpp文件就有多少个翻译单元。

#### 头文件的缺点？

1. 头文件被多少个翻译单元include，那么头文件中的内容就会被编译多少次
2. 较早被包含的头文件可能会影响后面被包含的头文件中的代码含义
3. 不一致性，可能会出现一个声明有不同的定义
4. include包含头文件，会拷贝头文件所有内容到翻译单元中，可能会暴露实现细节以及会导致代码的膨胀

#### 为什么还是在用头文件呢？

历史原因，重构一个大项目太麻烦了！

### 3.2.2 Modules

#### module的优点？

1. module只会编译一次，所有使用它的翻译单元共享这个module
2. 同一个翻译单元的多个module不会互相影响
3. module内部import或者include的内容不会传递到用户，也就是说没有传递性 
4. 性能都得到了很大的提升，因为不需要把预处理后的头文件内容拷贝到翻译单元再次编译了，module只需要编译一次。

ps：因为module没有传递性，所以对于代码量很小的情况，声明和实现在一个文件也可以。

#### 如何在模块中include？

全局模块片段，注意这样include的头文件不会污染用户代码。

```python
module;

#include ...

export module ...
```

## 3.3 Namespaces

### namespace的优点？

将声明限制在当前命令空间中

### using-directive的优缺点？

- 优点：可以不使用namespace修饰符前缀，减少代码冗余
- 缺点：丧失选择性使用这个namespace的声明的能力，因为默认就用了这个namespace。

只在app中频繁出现的库，或者修改一个老项目且这个老项目没有使用namespace的情况下使用using-direcitve（后者只是临时方案，为了减少代码修改）。

## 3.4 Function Arguments and Return Values

### 函数调用？

目的：程序不同部分之间传递信息

参数：执行一个任务所需要的信息

返回值：任务的结果

### 3.4.1 Argument Passing

#### 参数传递？

1. pass-by-value：默认，一般对于小于2个指针大小的对象都可以用
2. pass-by-reference：节省性能
3. pass-by-const-reference：节省性能又避免修改原值
4. default value：避免无意义的重载

### 3.4.2 Value Return

#### 函数返回值？

1. return-a-value：默认。返回局部变量不用担心性能，所以有移动拷贝会调用，如果没有编译器也会优化，直接在需要它的位置构造。
2. return-a-reference：返回引用或指针，永远不要返回局部变量的引用或指针

注意：

- 如果不能执行完所有流程，抛出异常。
- 不要手动去管理内存，比如返回一个裸指针指向手动分配的内存，然后交给用户delete

### 3.4.3 Return Type Deduction

#### 返回值类型推导？

`auto mul(int i, double d) { return i*d; }`

通过函数结果推导出返回值。

注意：使用时需要额外注意，因为修改实现会导致函数签名的改变

## 3.4.4 Suffix Return Type

#### 后缀返回类型的作用？

`auto mul(int i, double d) -> double { return i*d; } `

1. 可以通过参数或者返回值来推导类型，类似Return Type Deduction
2. 还可以对齐函数签名（c++之父很喜欢这个，但是历史原因，一般还是用前置返回类型吧）

### 3.4.5 Structured Binding

#### 同时绑定多个值？

```c++
std::map<int, int> map{{1, 2}};
for (const auto& [key, value] : map) {
    std::cout << key << ": " << value << std::endl;
}
```

只有public成员才能使用，private成员需要搞一些骚操作才行。

# 4 Error Handling

## 4.2 Exceptions

### 异常？

何时使用：当程序出错了，比如数组越界，不应该继续往下执行时，throw异常

机制：异常会unwind栈调用，一直向callerunwind，直到找到一个处理异常的函数。

最佳实践：不要过多使用try-catch，而是利用raii释放资源。只在可以决策程序应该在异常发生后怎么执行的位置catch异常。往往这是在业务层决定的，比如

```c++
void fetchUserData() {
    try {
        return callRemoteService();
    } catch (const TimeoutException& e) {
        // 这里就是决策点：我知道该怎么做，我决定重试
        return retryService(3); 
    }
}
```

## 4.3 Invariants

### 类的invariant？

- 构造函数负责来检查preconditions，然后构造invariant。比如vector的invariant就是一个连续容器，然后preconditions就包括传入的数组大小不能是负数。
- 成员函数负责维持这个invariant。

其实编程的语境下，任务事物都有invariant，调用者只需要确保precondition满足，就可以去使用它，而实现着就需要在precondition不满足时，做出对应的错误处理。

```c++
Vector::Vector(int s)
{
    // precondition
    if (s < 0) {
        throw length_error{"Vector constructor: negative size"};
    }
    
    elem = new double[s];
    sz = s;
}
```



### catch内部应该做什么？

当一个函数throw异常后，在函数内部没有使用raii的情况下，调用者需要在catch内部回收资源，如果没有决策权，那么就再次throw这个异常给上层。

```c++
void test(int n)
{
    try {
    	Vector v(n);
    }
    catch (std::length_error&) { // do something and rethrow
        cerr << "test failed: length error\n";
        throw; // rethrow
    }
    catch (std::bad_alloc&) { // ouch! this program is not designed to handle me
        std::terminate(); // terminate the program
    }
}
```

## 4.4 Error-Handling Alternatives

# 5 Classes

## 5.1 Introduction

### 5.1.1 Classes

#### 类是什么？

类是用户定义的类型，用于表达程序中的entity（作为一个独立的部分）。

比如：一个idea，entity，数据集合，都可以尝试放到一个类里面，因为他们都是独立的一个整体。

## 5.2 Concrete Types

#### concrete class是什么？

concrete class的行为类似于内置类型，对于用于而言这个类就是内置类型。他们的representation是definition的一部分。

#### 灵活度优化

- 问题：具体类的实现改变但接口不改变时，仍然会导致用户代码重新编译。 核心点在于接口任然和实现耦合了。
- 优化：使用资源句柄（一般为指针或引用），从而指向在free store的资源，具体类内部的实现是相对固定的，将大部分的实现放到了free store区，这也是stl中的常见实现方式。

### 5.2.1 An Arithmetic Type

#### complex类解析

1. 简单操作需要inline，避免无意义的函数调用
2. 需要在编译时间直接计算结果时，在函数签名添加constexpr
3. 默认构造函数可以避免未初始化问题
4. const函数不会修改对象，无论什么对象都可以调用，但是普通函数只有not-const对象可以调用
5. 尽量使用已有实现来实现新功能，比如实现了complex的==，那么!=就可以使用==的结果取反来实现。
6. 类内部的实现可以直接依赖具体实现，而不是只能使用对外提供的接口，从而确保了效率。注意这并不是破坏封装，因为类的实现者本身就是内部成员，依赖具体实现细节是没有问题的。

### 5.2.2 A Container

#### 容器是什么？

容器是持有元素集合的对象。

#### RAII？

- Resource Acquisition Is Initialization，资源获取即初始化。

- 简单来说，就是通过构造函数创建内存并初始化，然后通过析构函数销毁对象并释放内存。对于用户而言无需关心内存，对象出作用域时内存会被自动释放。

### 5.2.3 Initializing Containers

#### 往容器中添加元素？

- Initializer-list constructor：初始化一列表的元素到一个新容器中`Vector(std::initializer_list<double>);` 。当用户使用{}时其实就是创建了一个初始化列表
- push_back（类比于添加元素的接口）：往已存在的容器中添加元素

#### 类型转换的使用技巧？

减少使用类型转换，尽可能将类型信息掌握在编译器手中，比如使用模版等技术。

## 5.3 Abstract Types

#### 抽象类型？

隔离用户和实现细节，完全将接口和representation解耦。

- 基类负责接口声明，子类负责具体的实现
- 用户代码只依赖于基类，无论子类怎么变更，用户代码都无需重新编译。

缺点：必须使用指针或引用来管理对象。

## 5.4 Virtual Functions

#### 虚函数怎么知道应该调用哪个实现？

每个虚类都有一个虚函数指针和虚函数表（vtbl），虚函数指针指向了虚函数表，虚函数表里面就是不同函数的索引。

## 5.5 Class Hierarchies

#### 虚析构的必要性

用户使用的是基类指针，如果析构函数不是虚的，那么就无法正确调用到具体实现的析构函数，这就会导致资源泄露 。

### 5.5.1 Benefits from Hierarchies

#### 继承分类

- 接口继承：派生类可以完全替代基类，从而实现多态

- 实现继承：派生类复用基类的实现，从而简化派生类的实现

一般来说，只有明确是`is a`的关系才会使用集成，对于实现继承减少使用，通过组合来实现这种方式是更好的。

### 5.5.2 Hierarchy Navigation

#### dynamic_cast

- 使用场景：用户传递派生类到一个接受基类的接口，然后这个接口又要返回这个基类指针，用户明确知道当前类型一定是某个派生类，所以会将其转换成派生类使用。（向下转换）

注意：除了上述情况，一般减少使用，因为大量使用dynamic_cast就代表丢失了抽象的灵活性。

### 5.5.3 Avoiding Resource Leaks

#### 裸指针的问题

裸指针无法表达资源的拥有权，通过智能指针来解决。特别的，当需要释放资源时，必须使用智能指针，不然就是error-prove

# 6 Essential Operations

## 6.1 Introduction

### 6.1.1 Essential Operations

#### 类的essential operations

1. 默认构造
2. 带参构造
3. 拷贝构造
4. 移动构造
5. 拷贝赋值
6. 移动赋值
7. 析构

#### Rule of Zero

要么定义全部的essential operations，要么都使用默认

#### 显示默认或者不要

default：显示表示需要编译器的默认实现版本

delete：显示表示编译器不要默认生成。一般在基类中使用较多，比如delete拷贝构造/赋值，因为基类作为接口类本身就没有必要拷贝和赋值，而且基类并不是具体实现，从内存模型的角度只是派生类的一部分，进行copy会出问题。

### 6.1.2 Conversions

#### 单个参数的构造函数隐式转换问题

为了避免某个变量隐式转换成了我的对象类型，对单个参数的构造函数添加explicit表示必须显示转换。

```c++
// 3.14隐式转换成了complex对象
complex z1 = 3.14; // z1 becomes {3.14,0.0}
```

### 6.1.3 Member Initializers

#### default member initializers

当需要给某个成员添加默认值时使用，可以避免在每个构造函数中都添加对应的默认初始化，从而减少成员未初始化的情况。

```c++
class complex {
    double re = 0;
    double im = 0; // representation: two doubles with default value 0.0
public:
    complex(double r, double i) :re{r}, im{i} {} // construct complex from two scalars:{r,i}
    complex(double r) :re{r} {} // construct complex from one scalars:{r,0}
    complex() {} // default complex: {0,0}
    // ...
}
```

## 6.2 Copy and Move

### 6.2.1 Copying Containers

#### 何时定义拷贝函数？

在类内部，有指针负责管理一些对象时，默认的memberwise拷贝就不可以了，因为会导致多个指针指向同一个内存地址。

修改：发生拷贝时，创建一段新内存，然后将旧值拷贝到新内存中。对于拷贝赋值，还需要释放原有的内存，然后将当前指针指向新内存。

### 6.2.2 Moving Containers

#### 移动函数？

不考虑编译器优化的情况下，一个对象从函数返回时，会导致一次拷贝，但是这次拷贝是毫无意义的，因为函数内部的这个局部对象出函数就会被析构，所以这时就可以使用移动构造函数，将局部变量移动到调用者，从而提升效率。

#### move

有时，程序员明确知道当前值不再被使用，但是编译器没有这么聪明，所以通过std::move进行右值类型转换，从而实现移动操作。

注意：move没有真正进行移动操作，只是做了一个右值转换，函数返回一个右值引用。

## 6.3 Resource Management

### c++为何没有垃圾回收

- 垃圾回收在分布式场景下会有性能损耗，即使通过优良的实现可以减少，但是c++追求的是极致性能
- 资源不只是内存，还有文件、网络socket等，c++需要的是一个通用的资源管理技术，在确保不浪费额外资源的前提下完成对资源的回收
- 通过RAII搭配error handling可以有效管理资源，并且将对象资源绑定在一个作用域中，如果某些资源需要出作用域也可使用共享、移动等技术实现

## 6.4 Operator Overloading

### 操作符重载

- 操作符重载需要符合传统含义，比如`+`就是表示加法，从而避免歧义。
- 推荐将操作符重载实现为free-standing函数（类外但是在类的namespace中），从而操作符两侧的操作数可以被相同对待。

## 6.5 Conventional Operations

### 6.5.1 Comparisons (Relational Operators)

#### 不同操作符之间的关联性

不同操作符之间可能存在一个关联性，比如满足`==`，那么不等于就可以用`!(==)`表示。除了`<=>`，其他或多或少都有一定的关联操作符。

#### <=>

三元操作运算符，可以避免自定义过多关系运算符，只需要这一个就可以完成其他关系运算符的操作。返回值：

- `== 0`，等于
- `< 0`， 小于
- ` > 0`, 大于

注意：

- 无论类的成员有多少个，编译器都可以默认生成一个`<=>`重载，默认按照成员声明顺序比较。（必须手动=default才能默认生成，否则是不会生成的）
- 如果自定义了`<=>`，那么也必须自定义`==`，其他运算符编译器仍然会默认生成

### 6.5.2 Container Operations

#### 迭代器

迭代器类似于指针，指向了容器中的元素。迭代器的泛化和效率更高，因为用户完全不需要知道容器底层是什么类型。

- range-for也是是使用迭代器实现的。
- 对于const容器，使用const迭代器，比如cbegin, cend等

### 6.5.3 Iterators and “smart pointers”

#### 迭代器和指针的关系

- 可以把迭代器类比成指针，指针支持的操作，迭代器也有。

- 指针其实就是天然的迭代器，迭代器支持的操作，指针天生就支持了。

### 6.5.4 Input and Output Operations

#### `<<`和`>>`

- `<<`表示左移，iostreams中表示输出操作符
- `>>`表示右移，iostreams中表示输入操作符

### 6.5.5 swap()

#### swap

对swap的要求：

- 性能
- 不会抛出异常，避免数据状态被损坏（noexcept）

c++11后使用三次移动实现swap()，减少了不必要的拷贝，标准库比如sort就使用了swap来实现。

对于自定义类：如果拷贝耗时且支持移动，那么自定义移动操作，再按照需求自定义swap函数。

### 6.5.6 hash<>

#### 如何将自定义类放入到哈希表中

- 在std作用域下特化hash<>函数
- 自定义哈希器

在上述基础上，自定义类必须重载`==`操作符

## 6.6 User-Defined Literals

### 自定义字面值

字面值操作符（`""`）会将参数抓换成返回值，比如`"hello"s`表示string

`<return type> operator""<suffix>(<params>)`

# 7 Templates

## 7.1 Introduction

### 模版

模版是类或者函数，参数化了类型或者值。

## 7.2 Parameterized Types

### template和concept在概念层面的区别

- template：对于所有的类型T
- concept ：对于所有的类型T，使其满足P(T）

### template的实例化

模版加上模版参数即可实例化，在编译过程的末尾的实例时间进行实例化操作（也叫特化），比如`std::vector<int> vec`

### 7.2.1 Constrained Template Arguments

#### 被限制的模版参数优点

使用concept就是被限制的模版参数，否则就是不被限制的模版参数（传统的typename/class）。

- 可以在编译的早期就提示错误，且错误更加清晰
- 不被限制的模版参数，只有在实例后编译器才能发现错误，且晦涩难懂

### 7.2.2 Value Template Arguments

#### 值类型参数

`template<int N>`

注意：对于字符串字面值，无法直接当做值模版参数，可以使用`char*`来实现，如

```c++
template<char* s>
void outs() { cout << s; }

// 注意：值模版参数不能用栈对象来实例化，因为局部对象地址在运行时才确定
char arr[] = "Weird workaround!";

void use()
{
    outs<"straightforward use">(); // error (for now)
    outs<arr>(); // writes: Weird workaround!
}
```

### 7.2.3 Template Argument Deduction

#### 模版参数推导

编译器可以通过初始化值来推导出模板参数，注意C风格字符串是`const char*`类型而不是string类型。

#### 迭代器类型推导

```c++
template<typename T>
class Vector {
public:
    Vector(initializer_list<T>);
    
    template<typename Iter>
    Vector(Iter b, Iter e);
}
```

在没有推导指导的情况下，`Vector v2(v1.begin(), v1.begin() + 2);`，编译器会推导Iter

为`std::vector<int>::iterator`，但是编译器不知道T是什么，所以编译器最后保守将T推导为Iter本身。后续我通过v2[0]访问元素时，得到的就是一个迭代器，从而引发一堆奇怪的报错，让人直接摸不着头脑。

解决方案：

- 推导指导`template<typename Iter> Vector(Iter,Iter) -> Vector<typename Iter::value_type>`，相当于指定了T为Iter::value_type

- 更好的选择是使用concept

  ```c++
  template<std::input_iterator Iter>   // 只有真正的迭代器才能匹配
  Vector(Iter b, Iter e) {
      std::cout << "迭代器区间版本（受约束）\n";
      for (; b != e; ++b) push_back(*b);
  }
  ```

- 直接显式指定模板参数

## 7.3 Parameterized Operations

#### 实现被类型或值参数化的操作

1. 函数模版
2. 函数对象
3. lambda

### 7.3.1 Function Templates

#### 函数模版

如下sum将容器和初始值类型参数化了

```c++
template<typename Sequence, typename Value>
Value sum(const Sequence& s, Value v)
{
    for (auto x : s)
    	v+=x;
    return v;
}
```

注意函数模板不能是虚函数，因为编译器无法在编译器知道当前模版会有多少个实例（模版实例化发生在调用它的地方），所以无法创建vtbl

### 7.3.2 Function Objects

#### 函数对象（仿函数）

重载`()`运算符，从而可以像调用函数一样使用对象。

优点：函数对象可以携带状态，而普通函数不行。比如实现`>10 >20`，普通函数需要两个实现，但是函数对象就可以把数组直接保存为成员，然后生成不同对象去调用，但是使用的是同一套逻辑。

### 7.3.3 Lambda Expressions

#### lambda表达式

其实就是隐式生成一个函数对象

捕获用法：

- `[&]`引用捕获所有当前可访问的变量
- `[&a]`引用捕获a
- `[=]`值捕获所有当前可访问的变量
- `[a]`值捕获a
- `[this]`引用捕获this
- `[*this]`值捕获this
- 可以混合使用，比如[&a, b]

#### 7.3.3.1 Lamdas as function arguments

##### 使用lamda作为函数参数

使用场合：对于一些简单的场合使用lambda，复杂场景下仍然使用函数对象，从而提供更好的可读性和复用性。

```c++
template<typename C, typename Oper>
void for_each(C& c, Oper op) // assume that C is a container of pointers (see al
{
    for (auto& x : c)
    	op(x); // pass op() a reference to each element pointed to
}
for_each(v,[](unique_ptr<Shape>& ps){ ps->draw(); }); // draw_all()
for_each(v,[](unique_ptr<Shape>& ps){ ps->rotate(45); }); // rotate_all(45)
```

##### 泛型lambda

比如s是必须指针类型，且指向一个对象，所以可以使用concept来约束，从而提供更好的编译报错信息。

注意：concept也可以对auto使用的，且必须添加auto，因为concept是一个类型特征，后面必须有实际的类型才行。

```c++
for_each(v,[](Pointer_to_class auto& s){ s->rotate(r); s->draw(); });
```

#### 7.3.3.2 Lambdas for initialization

##### 使用lambda做初始化

背景：当对象初始化比较复杂时，比如需要根据不同情况初始化不同的值，原始做法是，先初始化对象然后switch/if赋值对应的值。

原始做法的问题：

1. 这根本不是初始化，而是赋值
2. 赋值分支过多，且都是对同一个对象处理，很容易忘记某个分支，导致使用脏数据，且编译不会提示错误

解决：将初始化代码放到lambda函数中，从而将每个分支的作用域独立开，然后直接返回对应的结果。独立作用域指的是，原始做法都会赋值同一个对象，但是lambda做法只是返回一个对象，所以不同分支互不影响。

C++哲学：尽可能在编译期就发现问题！

#### 7.3.3.3 Finally

##### 析构函数的作用

RAII的核心函数，提供了通用的清理资源的方式，出作用对象会自动调用析构函数

##### C函数返回的资源如何清理

问题：C函数比如malloc返回的资源不在一个对象中，无法通过RAII自动回收资源。

解决：创建一个函数，函数接收一个负责清理对象的函数对象，返回一个对象，这个对象被析构时就会自动调用这个清理函数了。

```c++
// 这个对象必须被调用者接收，所以申明为nodiscard
template <class F>
[[nodiscard]] auto finally(F f)
{
	return Final_action{f};
}

// 内部存储了这个函数对象
template <class F>
struct Final_action {
    explicit Final_action(F f) :act(f) {}
    ~Final_action() { act(); }
    F act;
};
```

##### 为什么不直接创建Final_action对象

1. 意图表达更加明确
2. 强制`[[nodiscard]]` 可以防止用户忽略这个清理动作
3. 接口统一，后续可以统一做修改，比如做类型擦除等

c++哲学：Self-documenting Code。

## 7.4 Template Mechanisms

### 7.4.1 Variable Templates

#### 变量模版

变量模版常被用来定义一个常量(constexpr)，注意现代C++中变量是指**有名字的存储单元**，而不是会变化的事物。（历史原因，仍然沿用变量的说法）

```c++
template <class T>
constexpr T viscosity = 0.4;
```

变量模版可以配合static_cast一起使用，从而在编译期做很多操作，如下在编译时间就能判断当前类型是否可赋值，这就是concept的核心理念。

```c++
template<typename T, typename T2>
constexpr bool Assignable = is_assignable<T&,T2>::value;

template<typename T>
void testing()
{
    static_assert(Assignable<T&,double>, "can't assign a double to a T");
    static_assert(Assignable<T&,string>, "can't assign a string to a T");
}
```

### 7.4.2 Aliases

#### 别名

using的优点：

- 提供了一层抽象。
- 可以用于给模版起别名（typdef不可以）。

using允许用户编写可移植代码，即使实现不同。例子：

- size_t的具体类型是依赖于实现的，但是size_t是一个别名，用户只需要知道这是一个size_t无符号类型即可。

- stl容器都有个value_type表示当前容器的元素类型，其实实现就是`using value_type = T`

- ```c++
  // 创建一个key永远是string的map
  template<typename Value>
  using String_map = Map<string,Value>;
  String_map<int> m;
  ```

### 7.4.3 Compile-Time if

#### 操作有多种实现时的处理

1. 虚函数，多态调用，然后用户根据条件选择对应的派生类实现（但是会有运行时开销）。
2. `if constexpr()`在编译期就进入对应的分支，编译器只会检查满足条件的分支，所有判断都在编译期决定，没有任何运行时的额外开销。

注意：`if constexpr`是一个语法关键字，而不是`#if`这种文本替换。

# 8 Concepts and Generic Programming

## 8.1 Introduction

### 模版的优点

1. 支持传递类型作为参数，从而提供了很高的灵活度和内联机会
2. 提供机会在实例期组合不同上下文的信息，从而让编译期进行优化
3. 支持传递值作为参数，从而提供在编译期计算的能力

template+concept是现代C++泛型编程的核心，也通过了模版参数化多态的能力（编译时多态）。

## 8.2 Concepts

### concept

当使用模版时，传入的类型需要满足指定的要求，这个要求就叫做concept。如下Seq必须是一个支持begin end的容器，Value必须是一个支持+=的元素。

```c++
template<typename Seq, typename Value>
Value sum(Seq s, Value v)
{
    for (const auto& x : s)
    	v+=x;
    return v;
}
```

### 8.2.1 Use of Concepts

#### 模版参数关系的concept

模版参数往往具有某种关系，约束这种关系可以提早发现问题，如下表达等价

```c++
// 个人喜欢这个，因为把参数concept和参数关系concept分开表达了
template<Sequence Seq, Number Num>
requires Arithmetic<range_value_t<Seq>,Num>
Num sum(Seq s, Num n);

// 过于冗余
template<typename Seq, typename Num>
requires Sequence<Seq> && Number<Num> && Arithmetic<range_value_t<Seq>,Num>
Num sum(Seq s, Num n);

// 把参数关系concept绑定到Num的concept中了
template<Sequence Seq, Arithmetic<range_value_t<Seq>> Num>
Num sum(Seq s, Num n);

// 如果不用concept，就只能在注释中表示concept了
template<typename Sequence, typename Number>
// requires Arithmetic<range_value_t<Sequence>,Number>
Number sum(Sequence s, Number n);
```

### 8.2.2 Concept-based Overloading

#### 基于concept的重载

在编译期就可以确定函数地址，所以没有运行时开销（虚函数会有），重载规则和普通函数重载差不多，只不多重载的是模版参数，换句话说就是重载的是类型。

```c++
template<forward_iterator Iter>
void advance(Iter p, int n) // move p n elements forward
{
	while(n--)
        ++p;
}

template<random_access_iterator Iter>
void advance(Iter p, int n) // move p n elements forward
{
    p+=n; // a random-access iterator has +=
}
```

### 8.2.3 Valid Code

#### requires requires

在不使用concept的情况下，也可以通过requires requires建立一个concept。

```c++
// 后者的requires(Iter p, int i) { p[i]; p+i; }是requires表达式
template<forward_iterator Iter>
requires requires(Iter p, int i) { p[i]; p+i; }
```

缺点：

- 缺乏灵活性、可读性、复用性，所以不要这么写代码。
- 核心在于当用户使用这个concept时，用户不需要查看你的实现就应该知道如何使用，但是requires requires违背了这个理念。

建议：首先使用requires表达式创建concept，用户直接使用concept，如同8.2.1章节的例子。

### 8.2.4 Definition of Concepts

#### 定义concept

- 无约束的concept，

  ```c++
  template<typename T>
  concept C = requires(T a, T b) {
      a + b;   // 只要 a+b 能编译通过即可
  };
  ```

- 有约束的concept，`{} ->`后面必须接concept

  ```c++
  // b是默认模版参数
  template<typename T>
  concept C = requires(T a, T b = a) {
      { a + b } -> std::integral;   // a+b 的结果必须是整数类型
  };
  ```

#### concept别名

```c++
// 容器
template<typename S>
concept Sequence = input_range<S>; 

// 容器值的类型
template<class S>
using Value_type = typename S::value_type;
```

前者更加通用，因为不是所有容器都有value_type这个类型，比如C数组或者自定义容器就没有。

#### 8.2.4.1 Definition Checking

##### concept检查什么

concept只检查接口，不检查实现。比如cmp，只检查T支持`==`，具体实现的检查要等待实例期过后才会检查。

```c++
template<equality_comparable T>
bool cmp(T a, T b)
{
	return a<b;
}
```

注意：C++中对象的实例化发生在当前对象被调用的时候。也就是说，没有用户调用这个对象，那么就不会被实例化。

模版定义的检查延迟到实例期的好处:

- 开发过程中可以使用不完整的concept，然后后续慢慢补充
- 在不影响接口的情况下，可以任意改变实现，从而避免了大量的文件重编译。（这是新手误区，因为模版实现在头文件中，就以为修改了模版实现也会导致大量的重编译）

### 8.2.5 Concepts and auto

#### auto和模版的关系

将auto理解为T，它就是一个类型，表示不约束的模版类型。

#### 约束auto

既然auto可以理解为模版，那么就可以对其使用concept。比如函数返回一个数值类型，如果直接使用auto等报错时，就是一大堆奇怪提示，使用concept将其约束。

```c++
Number auto some_function(int x)
{
    // ...
    return fct(x); // an error unless fct(x) returns a Number
    // ...
}
```

个人理解：auto几乎就等于模版参数

### 8.2.6 Concepts and Types

#### concept和type的比较

1. type和concept都可以表示支持一系列操作
2. type指定了内存布局，但是concept不关心布局，给了编译器更高的灵活度
3. type只能表示单一类型，concept表示一系列类型
4. type无法表达多参数之间的关系，但是concept可以

总结：concept表述的是接口规范，而type描述的是物理存储规范。前者更加通用且抽象层面更高。

建议：大量使用concept，业务层可能模版会少一些，但是谁用C++写业务呢？

## 8.3 Generic Programming

### 8.3.1 Use of Concepts

#### 正确定义concept

- 不应该让concept仅仅约束语法，而是约束语义。因为编译器可以检查语法，但是语义只能让程序员检查。

- 让concept约束领域相关的概念，比如加法就应该满足结合律。

```c++
// ❌ 语义无意义，只是支持加法，但是满足什么数学性质呢？
template<typename T>
concept Addable = requires(T a, T b) { a + b; };

// ✅ 语义明确（半群：满足结合律）
template<typename T>
concept Semigroup = requires(T a, T b, T c) {
    { a + b } -> std::same_as<T>;
    // 隐含： (a + b) + c == a + (b + c) （程序员必须保证）
};
```

备注：所以说单纯学好C++没有意义，C++必须搭配某一个领域才能大放异彩。

### 8.3.2 Abstraction Using Templates

#### 开发模版的流程

1. 首先书写具体版本
2. 调试、测试
3. 最后使用模版替换具体版本

除非肯定未来没有其他类型使用这个函数，否则哪怕只有一点点机会，都应该将其写成模版。原因：

1. 没有任何性能损耗
2. 代码通用性提高了很多
3. 优化成模版的复杂度也不高

## 8.4 Variadic Templates

### 可变参模版

模版参数可以是任意数量，极大提高了程序的灵活度，甚至可以使用递归。

```c++
// 打印任意数量的值
template<typename T>
concept Printable = requires(T t) { std::cout << t; } // just one operation!
void print()
{
// what we do for no arguments: nothing
}

template<Printable T, Printable... Tail>
void print(T head, Tail... tail)
{
    cout << head << ' '; // first, what we do for the head
    print(tail...); // then, what we do for the tail
}

/* ==========等价============== */

// 使用if constexpr来避免声明单独的结束函数
template<Printable T, Printable... Tail>
void print(T head, Tail... tail)
{
    cout << head << ' ';
    if constexpr(sizeof...(tail)> 0)
    print(tail...);
}
```

注意：必须使用`if constexpr`，因为在没有参数的情况，会调用`print()`，但是根本没有这个函数，所以使用普通if，编译器会报错，但是if constexpr根本不会检查判断会错的分支，所以没有问题。

缺点：正确实现太复杂了，且编译时间和空间开销线性增加。（书中写了很多缺点，我觉得就这两个就已经很恐怖了）

### 8.4.1 Fold Expressions

#### 折叠表达式

使用折叠表达式减少可变参模板递归的使用。

- 左折叠

  ```c++
  template<Number... T>
  int sum2(T... v)
  {
      return (0 + ... + v); // add all elements of v to 0
  }
  ```

  

- 右折叠

  ```c++
  template<Number... T>
  int sum(T... v)
  {
  return (v + ... + 0); // add all elements of v starting with 0
  }
  ```

### 8.4.2 Forwarding Arguments

假设有一系列对象的构造函数参数不同，可以使用可变参模版，然后通过完美转发将参数传递给他们各自的构造函数，这样子就实现了构造的通用化。如下用户只需要调用InputChannel来构造即可：

```c++
template<concepts::InputTransport Transport>
class InputChannel {
public:
    // ...
    InputChannel(Transport::Args&&... transportArgs)
    : _transport(std::forward<TransportArgs>(transportArgs)...)
    {}
    // ...
    Transport _transport;
};
```

优点：

- 相对比直接构造几乎没有额外开销（最多一个可以忽略的函数开销，而且这个开销还可能被内联优化掉）
- 通用接口

## 8.5 Template Compilation Model

#### 模版在编译时间的问题

- 模版的声明和定义必须在同一作用域中，当用户include模版的头文件就会把模版的定义一起包含进来。这就导致很容易大规模重新编译。
- 使用module来优化，因为module可以分离声明和实现，被半编译一个表示从而快速import和使用，而用户只需看到声明，编译速度大幅提升了。

# 9 Library Overview

## 9.1 Introduction

## 9.2 Standard-Library Components

### C++对于标准库组件的标准

- 对每个程序都有可能有用
- 通用版本比简单版本几乎没有大开销
- 很容易学习简单的使用

## 9.3 Standard-Library Organization

### 使用标准库的方式

标准库在namespace std里面，通过模块或头文件包含进来

### 9.3.1 Namespaces

#### 子命名空间

C++为了防止名字污染，在std命名空间下又添加了很多子命名空间。特别是后缀表达式，比如`"hello"s`可以表示字符串，也可以表示秒，所以需要子namespace来区分，用户使用`using namepace`。

建议：个人自定义后缀表达式时，也最好声明一个子namespace。

### 9.3.2 The ranges namespace

#### range和std作用域的冲突

模版在推导参数类型时，很可能会把参数不同的函数，实例化成签名一样的，所以C++建议显示指定作用域。比如

```c++
// 表面看是重载了，但是模版被实例化后有可能签名就成一样的了
sort(v.begin(),v.end());
ranges::sort(v); 
```

### 9.3.3 Modules

#### 标准库模块

C++20没有标准库的模块，23好像有了。不过可以自己创建std的模块。

### 9.3.4 Headers

#### 头文件

C++头文件并没有那么多逻辑关联性，所以不好记，主要是历史原因，这也是为什么C++要开发模版的原因之一。

# 10 Strings and Regular Expressions

## 10.1 Introduction

## 10.2 Strings

### string的常用操作

1. 字符串连接：+, +=
2. 获取子字符串（拷贝）：substr
3. 替换：replace

### 10.2.1 string Implementation

#### short-string optimization

对于短字符串，直接将其保存到对象中，对于长字符串将其保存到free store中。

原因：

- 假设系统中有多个不同大小的字符串，全部保存到free store，很容易产生内存碎片。
- 在多线程系统重，分配内存开销更高

#### stl的string实现

- string只是`basic_string<char>`的别名
- 用户想要什么类型的元素，就可以去实例化，比如使用日本字符，就用`basic_string<Jchar>`。当然这里只是假设有Jchar而已。

## 10.3 String Views

### string_view

string_view本质上就是指针和长度对，用于表示一个字符序列。

背景：用户可能传std::string c-style string这种类型给函数，使用原始的string作为参数，就必然会遇到将c-style string转换为std::string，造成没有必要的构造。

解决：string_view只是简单的指针+长度，所以作为函数参数，即使是需要构造，开销也很小，几乎没有额外的性能损失。

建议：把string_view当做裸指针来认真对待，因为它必须要指向一个实际存在的对象，否则是未定义行为。

### string vs string_view

- string拥有字符序列所有权且可以读写
- string_view没有所有权，只是可以读取字符无法写

## 10.4 Regular Expressions

### 原始字符串字面值

`R"(...)"`内部不会发生转义

### 10.4.1 Searching

后面章节没看，用到再来看。

# 11 Input and Output

## 11.1 Introduction

## 11.2 Output

### 输出流

- 所有值都会被cout转换为一系列字符

- cout表达式的结果可以被用来继续接收输出

  ```c++
  cout << "the value of i is " << i << '\n';
  ```

## 11.3 Input

### 输入流

- 将字符串字面值转换成对应类型
- cout表达式的结果可以被用来继续接收输入，和cin一样

cin的结束：

- 读取到非当前需要的类型时终止
- 默认情况下，读取到一个空格或换行就会停止读取
- 为了读取有空格的字符串，需要使用std::getline()，getline会把换行读取并丢弃

## 11.4 I/O State

### io状态

- 将io放到if中，返回值会自动转换为ture或false，表示当前操作是否成功。
- 提供了很多状态类型，用来表示当前的具体状态，比如eof()函数表示是否到达末尾。

## 11.5 I/O of User-Defined Types

### 重载`<< >>`操作符

```c++
ostream& operator<<(ostream& os, const Entry& e);

istream& operator>>(istream& is, Entry& e);
```

## 11.6 Output Formatting

### iostream和format提供的功能区别

- iostream提供的是基于流的格式化打印
- format提供的是printf风格的值的组合打印

### 11.6.1 Stream Formatting

#### 操控符

通过操控符来实现格式化

```c++
#include <iostream>
#include <iomanip>   // 为了 setw 和 setfill

int main() {
    // 无参操控符（来自 <ios>，通过 <iostream> 间接包含）
    std::cout << std::hex << 255 << std::dec << '\n';  // ff

    // 有参操控符（来自 <iomanip>）
    std::cout << std::setw(10) << std::setfill('*') << 42 << '\n';
    // 输出: ********42

    return 0;
}
```



### 11.6.2 printf()-style Formatting























































# A Module std

## A.1 Introduction

### 导入std模版

C++20没有支持直接导入std模块，所以需要自己写std

### 导入全局作用域函数

C++为了和C兼容，在全局作用域放了一些函数，比如`::sqrt() `和std作用域下的`std::sqrt`，为了支持全局作用域函数，需要`import std.compat`

### 模块和头文件

- 模块不会导入宏，所以当需要使用宏时使用include。因为宏是文本级别，而module是编译单元级别，我们实际项目应该减少使用宏。
- include和module是可以共存的。

## A.2 Use What Your Implementation Offers

### 创建std模块（使用编译器实验性实现）

核心思想为：导入后马上导出

注意：因为一些编译器是实验性支持module，所以可能编译器版本更新会导致不一样的结果，虽然如下代码应该区别不大。而且只有vs2022现在支持。

```c++
export module std;
export import std.regex; // <regex>
export import std.filesystem; // <filesystem>
export import std.memory; // <memory>
export import std.threading; // <atomic>, <condition_variable>, <future>, <mutex>,
// <shared_mutex>, <thread>
export import std.core; // all the rest
```

## A.3 Use Headers

### 使用头文件的时机

不支持模版的情况下才使用，否则都别用。

## A.4 Make Your Own module std

### 自定义std模块（包含头文件来实现）

没看明白，但是一般也不用这个。

```c++
module;
#include <iostream>
#include<string>
#include<vector>
#include<list>
#include<memory>
#include<algorithms>
// ...
export module std;
export istream;
export ostream;
export iostream;
// ..
```

