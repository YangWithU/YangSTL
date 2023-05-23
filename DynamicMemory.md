# 12 - Dynamic Memory

## 计算机内存基本六大类

| 内存类型 | 名称 | 介绍 |
| --- | --- | --- |
| 静态内存 | Static memory | 在程序运行前就已经分配好，通常用于存储全局变量、静态变量、常量等，生命周期与程序运行周期相同 |
| 栈内存 | Stack memory | 在函数调用时动态分配，用于存储局部变量和函数调用的上下文信息，生命周期与函数调用周期相同 |
| 堆内存 | Heap memory | 程序员手动分配和释放的内存空间，一般用于动态创建对象或存储大量数据，生命周期由程序员管理 |
| 共享内存 | Shared memory | 可以被多个进程共享的内存空间，用于提高进程间通信的效率 |
| 虚拟内存 | Virtual memory | 一种抽象的内存概念，为应用程序提供了一个连续的地址空间，允许操作系统和应用程序使用比物理内存更大的内存空间。虚拟内存通过在磁盘上创建一个交换文件来实现，当物理内存不足时，将数据从物理内存移动到交换文件中，以便腾出内存空间 |
| 堆栈内存 | Hybrid memory | 将栈内存和堆内存结合使用的一种内存模型，用于提高内存的使用效率 |

## 栈内存，堆内存

堆内存是由程序员通过动态分配内存的方式手动分配和释放的内存空间，也就是堆内存的生命周期不受函数作用域的限制。所以在使用堆内存时，需要注意内存的管理和释放，避免出现内存泄漏和其他问题。

具体分配和释放操作为 `new` & `delete`。无论在何处，手动采用new总会在堆内存上创建新的内存，所以在生命周期结束后，必须手动释放。

栈内存是操作系统分配的局部内存，它在函数被调用时动态分配，而当函数结束时会自动释放内存。栈内存的生命周期是由编译器自动管理的，程序员无法手动控制。

main函数的内存也属于 **栈内存**

## static 对象

声明变量时，给变量类型加上 `static` 关键字后，就成了 `静态(static)` 对象。
此种对象并不在声明处作用域内分配内存，而是会与全局变量一起，在 `BSS` 段 或 `DATA 段` ，在 **静态内存区域** 分配内存。
全局变量和 `静态(static)` 对象的内存位于 `BSS段` 和 `DATA 段`，其中：

+ `BSS段` 存储未初始化的全局变量和静态(static)变量
+ `DATA 段` 存储已初始化的全局变量和静态(static)变量

### static 用途

综上，`static` 对象具有如下性质：

+ 在静态内存处初始化
+ 必须在函数外部，类外部进行显式声明赋值
+ 声明赋值时必须指定变量的作用域

通过以上的种种性质，我们可以总结出：

对于一个类中的 `static` 对象，它并非其内部亲生成员，类只是给了它一个作用域，情况对于类中的 `static` 函数同样相同。

对于函数中的 `static` 对象，我们可以利用它恒量恒定的性质，完成特殊操作。

### 辨析

#### quiz1::如何使用static

static的性质一：

```c++
int Foo() {
    static int x = 114514;
    return x++;
}
// main:
for(int i = 0; i < 3; i++) std::cout << Foo() << " ";
/* output: */
114515 114516 114517 
```

`static` 变量在全局变量处初始化，这意味着整个程序只会将该变量初始化一次

> 同时也就是说全局变量具有全局生命期，生命随程序结束而结束

```c++
class Foo {
public:
    static int num;
};

int Foo::num = 114514;

int main()
{
    cout << Foo::num << endl;
}
```

`static` 变量在全局变量处初始化，这意味着对于拥有static变量的对象，在该对象 `Foo` 被创建前， `static` 变量可能就已经被创建

此外，`static` 函数也是如此，可以利用它处理static成员变量：

```c++
class Foo {
public:
    static void opt() {
        cout << num << endl;
    }
private:
    static int num; // static变量赋值只能在class外单独赋值
};

int Foo::num = 114514;

int main()
{
    Foo::opt();
}
/* output */
114514
```

如上，我们用一个成员 `static` 函数去获取 `static` 成员变量的值，可以发现，我们并没有具象化class，而是直接访问静态内存，获取该 `static` 对象

## 为什么我们要使用动态内存

使用动态内存的目的并非为了节省内存，而是因为:

+ **栈内存变量受其作用域约束，一旦作用域结束立即释放**

> 因此除非使用栈内指针或分配到堆内存，否则栈变量超出作用域难逃deallocate

+ 静态内存与全局变量内存较为连续，大量操作会产生内存碎片降低效率
+ 动态分配的内存具有自由的生命周期，全由我们掌控

结合以上性质，我们可以认为堆内存可用于：

+ 在静态内存和栈内存之间充当 "信使"，传递数据
+ 自定义对象生命周期，用于处理满足相应特性的数据

> 假若我们开发"学生信息管理系统"，那么我们就可以将学生的信息存储到堆内存中，从而可以简易添加和释放相关数据

## new&delete: 直接管理堆内存

存在 `operator new` 和 `placement new` ，先讲 `operator new`

### operator new

usage:

```c++
int *numptr = new int(114514); // *numptr = 114514
string *strptr = new string(9, 'a'); // *strptr = "aaaaaaaaa"
vector<int> *vctptr = new vector<int> {0, 2, 4, 6, 8, 10};

// must dereference: (*vctptr) first
cout << (*vctptr)[1] << endl; // 2
```

可以发现两个性质：

+ `new` 分配出的这些变量没有名称，只有一个指向它的指针。在自由空间分配的变量是无名的
+ 默认时，`new` 会调用对象的默认 `ctor`，对于没有默认ctor的对象，其值未定义，除非自己手动值初始化
  
```c++
int *pi = new int; // undefined: not contain ctor
std::string *ps = new std::string; // *ps = "", using ctor

int ppi = new int(); // init to 0 manually
std::string *pps = new std::string(); // init to "" manually
```

### placement new

使用 `new` 时可能存在自由空间内存耗尽的情况，这时 `new` 会抛出 `std::bad_alloc` ，我们可以传入 `nothrow` 忽略异常，使 `new` 返回一个空指针。

```c++
int *p = new (nothrow) int;
```

### operator delete

`delete` 接受一个指针(可以是空指针)，将指针所指的动态内存归还，但需要注意：

+ **不可释放非 `new` 分配的内存,即静态内存**
+ **不可重复释放同一块内存**

Approach：

```c++
int a = 114;
int *p = &a;
delete p; // error
```

> 上述p是指向局部变量的指针，属于栈内存，因此不可用delete删除
> delete只能释放堆内存

```c++
int *a = new int(114);
int *p = a;
delete a;
std::cout << p;
```

此时 `p` 指针仍然指向一处已经被释放的内存 `a`，产生未定义行为

```c++
const int *a = new const int(114);
delete a;
```

最后，也可以在动态内存上分配 `const` 对象，除了不可修改对象值外，可以对此内存 `delete`

## 动态内存与智能指针

由上，动态分配内存时，我们将手动在堆内存上分配内存。为避免悬停指针，内存溢出等情况，我们可以采用智能指针，自动回收指针，解决上述问题

`c11` 提供两种智能指针形式： `shared_ptr` 和 `unique_ptr` , 此外还有一种 `weak_ptr` 的伴随类，指向 `shared_ptr` 所管理的对象。

intro:

```c++
struct Node {
    int x, y;
    Node(int _x, int _y) : x(_x), y(_y) {}
    Node() = default;
    void printall() {
        std::cout << x << " " << y << "\n";
    }
};

void Foolish() {
    Node* npt = new Node(114514, 1919810);

    npt->printall();
}

void smart() {
    // less perfered: separated allocation and construction
    // std::shared_ptr<Node> smpt(new Node(114514, 1919810));

    // perfered: all in one, one for all
    auto smpt = std::make_shared<Node>(Node(114514, 1919810));
    smpt -> printall();
}

int main()
{
    Foolish();
    //smart();
    //return 0;
}
```

通过 `valgrind` 分析发现:

> 8 bytes in 1 blocks are definitely lost in loss record 1 of 1

在我们的 `Foolish` 函数中发生内存泄漏，原因在于我们使用new在堆内存上分配后没有释放指针 `npt` ，造成内存泄漏，使得该处内存被持续占用。将该指针delete后问题解决。

同样是指向对象的指针，同样为指针对应内存分配了空间，但普通指针生命结束后内存未曾释放，智能指针则完美解决问题。

### 为什么要避免内存泄漏呢？

假设你为电梯管理系统写的程序存在内存泄漏
傍晚，你坐上电梯。运行了一天的电梯内存芯片因为程序存在内存泄漏，
此时它狭小的内存芯片被挤的满满的，甚至快溢出来了
终于，在你乘上电梯的时刻内存被完全填满，程序崩溃
你被困在漆黑的电梯中享受自己的杰作

采用智能指针，省去释放烦恼。

## 两种智能指针区别

`shared_ptr` 允许多个指针指向它指向的对象，但 `unique_ptr` 则与此不同

## 使用 `shared_ptr`

### 使用 `make_shared`

最安全分配使用和分配动态内存的方式是使用 `make_shared`
因为它同时集成 `内存分配` 和 `对象构造` 两步操作
它会返回一个指向制定对象的 `shared_ptr` 指针：

```c++
std::shared_ptr<int> p1 = std::make_shared<int>(114514);

// 未指定值时,调用string类默认ctor,赋默认值: ""
std::shared_ptr<std::string> p2 = std::make_shared<std::string>();

// 我们更倾向于使用auto自动推导类型,便捷
auto p3 = std::make_shared<std::string>("hello ptr!")
```

### `shared_ptr` 拷贝与赋值

每个 `shared_ptr` 都会记录有多少个其他的 `shared_ptr` 指向相同的对象，用于记录数量的计数器称为 `reference counter`

+ 拷贝 `shared_ptr` 时, `reference counter` 递增
+ 赋新值或销毁 `shared_ptr` 时, `reference counter` 递减

> 上述*拷贝*是指：
>
> + 将当前shared_ptr作为参数传递
> + 将当前shared_ptr作为函数返回值
> + 将当前shared_ptr去初始化另一个shared_ptr

当 `reference_counter` 为0时，表明此时没有 `shared_ptr` 指向对象，则调用 `shared_ptr` 的析构函数，将该对象 `delete`

### Example

```c++
struct Node {
    Node() {
        std::cout << "ctor was called" << "\n";
    }
    ~Node() {
        std::cout << "dtor was called" << "\n";
    }
};

void use_func() {
    std::shared_ptr<Node> p = std::make_shared<Node>();
    std::cout << "***" << std::endl;
} // till now, end of p lifetime, deleted
```

当我们调用函数 `use_func` 后，创建一个指向Node的 `shared_ptr` ， `Node` 的ctor被调用。当函数 `use_func` 结束后， 智能指针 `p` 离开作用域， `shared_ptr` 调用 `Node` 的dtor

### 拷贝辨析

Case1:

```c++
void use_func() {
    std::shared_ptr<Node> p = std::make_shared<Node>(Node());
    //Node* pr = new Node();
    std::cout << "***" << std::endl;
} // out of range, smartptr calls dtor
```

上述代码结果为：

```c++
ctor was called
dtor was called
***
dtor was called
```

发生什么了？
其实就是我们在 `make_shared` 的参数位置进行了 `Node` 的显式初始化。之后我们将已经初始化的对象的内存拷贝给新创建的智能指针 `p` ，然后调用当前对象的 `dtor`
**此后智能指针 `p` 的作用域结束， `reference counter` 递减为0**, 调用 `p` 对象的 `dtor`，释放它占用的内存。

与普通指针对比可以发现，普通指针仅仅在分配时调用ctor,之后new分配内存，不会有任何的检查

Case2:

```c++
std::shared_ptr<Node> use_func() {
    std::shared_ptr<Node> p = std::make_shared<Node>();
    return p;
}
```

上述代码返回一个指针 `p` 的**拷贝**，此时递增了智能指针 `p` 的 `reference counter` ，在该变量作用域结束后， 尝试销毁智能指针， `reference counter` 递减但不为0，所以仍然保留这一块内存

如果此时再忘记清理当前这块内存，就会造成浪费

Case3:

```c++
struct Node {
    int x;
    explicit Node(int v) : x(v) { std::cout << "ctor called" << "\n"; };
    ~Node() { std::cout << "dtor called" << "\n"; }
    void opt() const {
        std::cout << x << "\n";
    }
};

int main()
{
    std::shared_ptr<Node> p1(new Node(42));
    std::shared_ptr<Node> p2(new Node(100));
    std::cout << "ori p1:" << "\n";
    p1->opt();
    p1 = p2;
    std::cout << "now p1:" << "\n";
    p1->opt();
    std::cout << "p2:" << "\n";
    p2->opt();
}
```

在上述过程中，我们单独创建了两个不同的智能指针。
结果说明，当p2拷贝到p1时，p1调用dtor清空数据,之后拷贝p2数据到自身。

case4:

```c++
void foo(const std::shared_ptr<int> ptr) {
    std::cout << *ptr << "\n";
}

int main()
{
    // p reference counter: 1
    std::shared_ptr<int> p(new int(114));
    // p reference counter: 2
    foo(p);
    // now, p reference counter: 1
    // still able to reach:
    std::cout << *p << "\n";
    int *x = new int(514);

    // x reference counter: 1
    foo(std::shared_ptr<int>(x));
}
```

过程如上所示，可以比较发现二者的 `reference counter` 是不同的，`p` 仍然可以在foo之后再次获取此处内存，但 `x` 的 `reference counter` 已经递减为零。

> ps:  当我们调用 `shared_ptr` 或 `make_shared` 的时候在初始化中应当关注 `ctor` 的显式与隐式情况，声明自定义ctor后就必须显式初始化，此外如果类中只有单独的ctor，我们应当加上 `explicit` 关键字优化程序:

```c++
struct DFctor {
    int x;
};

struct EPctor {
    int x;
    explicit 
    EPctor(int v) : x(v) { }
};

auto p1 = std::make_shared<DFctor>(DFctor()); // default ctor
auto p2 = std::make_shared<EPctor>(EPctor(114));
```

### 使用智能指针共享对象之间数据

```c++
class StrTypo {
public:
    typedef std::vector<std::string>::size_type size_type;

    StrTypo() : vs_ptr(std::make_shared<std::vector<std::string>>()) { }
    StrTypo(std::initializer_list<std::string> il) :
            vs_ptr(std::make_shared<std::vector<std::string>>(il)) { }

    // Think: why const?
    size_type size() const { return vs_ptr->size(); }
    bool empty() const { return vs_ptr->empty(); }

    void push_back(const std::string& t) { vs_ptr->push_back(t); }
    void pop_back(const std::string& t);

    const std::string& front();
    const std::string& back();

private:
    // 调用标准库中string及vector的操作函数
    std::shared_ptr<std::vector<std::string>> vs_ptr;
    // Error handler
    void check(size_type tp, const std::string& msg) const;
};

void StrTypo::check(size_type tp, const std::string &msg) const {
    if(tp >= vs_ptr->size())
        throw std::out_of_range(msg);
}

const std::string& StrTypo::front() {
    check(0, "front op on empty StrTypo!");
    return vs_ptr->front();
}

const std::string& StrTypo::back() {
    check(0, "back op on empty StrTypo!");
    return vs_ptr->back();
}

void StrTypo::pop_back(const std::string &t) {
    check(0, "pop_back op on StrTypo!");
    vs_ptr->pop_back();
}
```

如何在不去拷贝数据的情况调用其他类中的数据呢？
答案是使用指针，在源数据中实现指针，直接拷贝该指针即可

```c++
void ops(std::shared_ptr<StrTypo>& p) {
    std::shared_ptr<StrTypo> q(p);
    q->push_back("new one");
}

int main()
{
    std::shared_ptr<StrTypo> p(new StrTypo({"114514", "1919810"}));
    ops(p);
    std::cout << p->back();
}
```

在上述代码中，我们创建一个包含一个智能指针作为成员的类，从而实现**多个同类，一处存储**的操作，使得内存的使用更加连续

### 使用 `get`

存在一种情况：当函数不接受智能指针，只要普通指针时，考虑使用智能指针的内置指针

```c++
std::shared_ptr<int> p(new int(114));
int *q = p.get();
```

#### 注意

需要注意，使用 `get` 获得的指针不能在智能指针本身被释放后继续使用:

```c++
int *q;
{
    std::shared_ptr<int> p(new int(114));
    q = p.get();
} //now p deleted
std::cout << *q; // undefined behavior
```

所以永远不要用 `get` 去初始化新的指针

### 使用 `reset`

哥哥，为了将智能指针分配的内存重新分配或改变，就对他使用 `reset` 吧。他会释放旧对象，更新引用计数

usage:

```c++
auto p = std::make_shared<std::vector<std::string>>();
*p = {"one", "two"};

auto q = p;
if(!p.unique()) // if p has copied
    p.reset(new std::vector<std::string>({"copied"}));

for(const auto& x : *p)
    std::cout << x << " ";
```

### 自定义 `dtor` 的情况

有时候我们需要自定义 `dtor` ，为了在顺利实现自定义需求的同时简化操作，考虑利用智能指针简化操作，将自定义dtor作为第二个参数传入：

```c++
struct Node {
    int *p;
    void deconstruct();
    // sometimes, dtor in our condition even don't exist
    ~Node() = delete;
};

void deconstruct(Node *ndp) {
    ndp->p = nullptr;
}

void end_func(Node* nd) {
    // will destruct using `deconstruct`
    std::shared_ptr<Node> p(nd, deconstruct);
    /* now deal with objects */
}
```

