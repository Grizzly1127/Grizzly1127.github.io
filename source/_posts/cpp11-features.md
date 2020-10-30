---
title: C++11新特性
date: 2017-07-22 15:13:19
tags:
- c++
- c++11
categories: 
- c++
copyright: true
---

在看《C++ Primer Plus》的时候，了解到了很多 C++11 的新特性，将其记录下来，一是能加深印象，二是日后随时随地都能方便查看。

## 1. 新功能

### 1.1 新类型

新增类型 long long 和 unsigned long long，以支持64位（或更宽）的整型；新增类型 char16_t 和 char32_t，以支持16位和32位的字符表示。

### 1.2 统一的初始化

<!-- more -->

C++11扩大了用大括号括起的列表（初始化列表）的适用范围，使其可用于所有内置类型和用户定义的类型（即类对象）。使用初始化列表时，可添加等号（=），也可不添加：

```c++
// 所有内置类型
int x = {5};
double y {2.42};
short quar[5] {4,5,2,12,1};
// 可用于new表达式
int * arr = new int [4] {2,4,6,7};
// 可用于自定义对象
class Stump {
private:
    int roots;
    double weight;
public:
    Stump(int r, double w) : roots(r), weight(w) {}
};

Stump s1(3, 15.6);    // old style
Stump s2{5, 43,4};    // C++11
Stump s3 = {4, 32.1}; // c++11
```

新增模板类 **initializer_list**，可将其用作构造函数的参数、常规函数的参数：

```c++
#include <initializer_list>
double sum(std::initializer_list<double> il);
int main() {
    double total = sum({2.5, 3.1, 4}); // 4 将转为4.0
    // todo: ...
}
double sum(std::initializer_list<double> il) {
    double tot = 0;
    for (auto p = il.begin(); p != il.end(); p++)
        tot += *p;
    return tot;
}
```

### 1.3 声明

#### 1.3.1 auto

以前，关键字 **auto** 是一个存储类型说明符，C++11 将其用于实现自动类型推断：

```c++
auto a = 112; // a is type int
auto pt = &a; // pt is type int *
double fm(double, int); // function
auto pf = fm; // pf is type double (*)(double, int)
```

#### 1.3.2 decltype

关键字 **decltype** 将变量的类型声明为表达式指定的类型：

```c++
double x;
int n;
decltype(x*n) q; // q same type as x*n, i.e., double
decltype(&x) pd; // pd same as &x, i.e., double *
```

#### 1.3.3 返回类型后置

C++11 新增了一种函数声明语法：在函数名和参数列表后面指定返回类型：

```c++
double f1(double, int); // old style
auto f2(double, int) -> double; // new syntax, return type is double
```

在模板函数中使用：

```c++
template<typename T, typename U>
auto eff(T t, U u) -> decltype(T*U) {
    // todo: ...
}
```

#### 1.3.4 模板别名：using =

对于冗长或复杂的标识符，如果能够创建其别名将很方便。以前，C++为此提供了 typedef：

```c++
typedef std::vector<std::string>::iterator itType;
```

C++11 提供了另一种创建别名的语法：

```c++
using itType = std::vector<std::string>::iterator;
```

差别在于，新语法也可用于模板部分具体化，但 typedef 不能：

```c++
template<typename T>
using arr12 = std::array<T, 12>;
```

#### 1.3.5 nullptr

空指针是不会指向有效数据的指针。以前，C++ 在源代码中使用 0 表示这种指针，但这带来了一些问题，因为这使得 0 即可表示指针常量，又可表示整型常量。
新增了关键字 nullptr，用于表示空指针，它是指针类型，不能转换为整型类型。为向后兼容，C++11 仍允许使用 0 表示空指针，因此表达式 nullptr == 0 为 true。

### 1.4 智能指针

C++11 摒弃了 auto_ptr，并新增了三种智能指针：**unique_ptr**、**shared_ptr** 和 **weak_ptr**，定义在 `<memory>` 中。
三者皆可对动态资源进行管理，保证任何情况下，已构造的对象最终会被销毁，也就是它的析构函数最终会被调用。

#### unique_ptr

unique_ptr 持有对象的独有权，同一时刻只能有一个 unique_ptr 指向给定对象（通过禁止拷贝语义，只有移动语义来实现）。
unique_ptr 指针本身的生命周期：从 unique_ptr 指针创建时开始，直到离开作用域。

```c++
#include <memory>
#include <iostream>

int main()
{
    std::unique_ptr<int> up1(new int(11));  // 无法拷贝
    // std::unique_ptr<int> up2 = up1;      // err, 编译时错误
    std::cout << *up1 << std::endl;         // 11

    std::unique_ptr<int> up3 = std::move(up1); // 可以移动，up3独占该指针
    // std::cout << *up1 << std::endl;         // err, 运行时错误
    std::cout << *up3 << std::endl;         // 11
    up1.reset();    // 不会导致运行时错误
    up3.reset();    // 显示释放内存
    // std::cout << *up3 << std::endl;         // err, 运行时错误

    std::unique_ptr<int> up4(new int(22));  // 无法拷贝
    up4.reset(new int(44)); // "绑定"动态对象
    std::cout << *up4 << std::endl;         // 44
    up4 = nullptr; // 显示销毁所指向对象，同时智能指针变为空指针，与reset()等价

    std::unique_ptr<int> up5(new int(55));
    int *p = up5.release(); //只是释放控制权，不会释放内存
    std::cout << *p << std::endl;   // 55
    //cout << *up5 << endl; // err, 运行时错误
    delete p; //释放堆区资源
    return 0;
}
```

#### shared_ptr

shared_ptr 允许多个该智能指针共享一段内存，通过引用计数（reference counting）实现，会记录有多少个 shared_ptr 共同指向一个对象，一旦最后一个 shared_ptr 被销毁，也就是某个对象的引用计数变为 0，这个对象会被自动删除。

```c++
#include <memory>
#include <iostream>

int main()
{
    std::shared_ptr<int> sp1(new int(11)); // 引用计数 = 1
    std::shared_ptr<int> sp2 = sp1; // 允许拷贝，引用计数加一
    std::cout << "sp1 count: " << sp1.use_count() << std::endl; // sp1 count: 2
    std::cout << "sp2 count: " << sp2.use_count() << std::endl; // sp2 count: 2
    std::cout << *sp1 << std::endl; // 11
    std::cout << *sp2 << std::endl; // 11

    sp1.reset(); // 显示释放对象控制权，引用计数减一
    std::cout << "sp2 count: " << sp2.use_count() << std::endl; // sp2 count: 1
    std::cout << *sp2 << std::endl; // 11
    return 0;
}
```

#### weak_ptr

weak_ptr 是为配合 shared_ptr 而引入的一种智能指针来协助 shared_ptr 工作，它可以从一个 shared_ptr 或另一个 weak_ptr 对象构造，它的构造和析构不会引起引用计数的增加或减少。没有重载 `*` 和 `->`，但可以使用 lock 获得一个可用的 shared_ptr 对象。

weak_ptr 的使用更为复杂一点，它可以指向 shared_ptr 指针指向的对象内存，却并不拥有该内存，而使用 weak_ptr 成员 lock，则可返回其指向内存的一个 share_ptr 对象，且在所指对象内存已经无效时，返回指针空值 nullptr。

注意：weak_ptr 并不拥有资源的所有权，所以不能直接使用资源。
可以从一个 weak_ptr 构造一个 shared_ptr 以取得共享资源的所有权。

```c++
#include <memory>
#include <iostream>

void check(std::weak_ptr<int>& wp)
{
    std::shared_ptr<int> sp = wp.lock();
    if (sp != nullptr) {
        std::cout << *sp << std::endl;
    } else {
        std::cout << "pointer is invalid" << std::endl;
    }
}

int main()
{
    std::shared_ptr<int> sp1(new int(11)); // 引用计数 = 1
    std::shared_ptr<int> sp2 = sp1; // 允许拷贝，引用计数加一
    std::weak_ptr<int> wp1 = sp1; // wp1的构造不会引起引用计数加一

    std::cout << "wp1 count: " << wp1.use_count() << std::endl; // wp1 count: 2
    std::cout << *sp1 << std::endl; // 11
    std::cout << *sp2 << std::endl; // 11
    check(wp1); // 11

    sp1.reset(); // 显示释放对象控制权，引用计数减一
    std::cout << "wp1 count: " << wp1.use_count() << std::endl; // wp1 count: 1
    std::cout << *sp2 << std::endl; // 11
    check(wp1); // 11

    sp2.reset(); // 显示释放对象控制权，引用计数减一
    std::cout << "wp1 count: " << wp1.use_count() << std::endl; // wp1 count: 0
    check(wp1); // pointer is invalid
    return 0;
}
```

### 1.5 异常规范方面的修改

C++11 提供了关键字 noexcept，该关键字告诉编译器，函数中不会发生异常,这有利于编译器对程序做更多的优化。

### 1.6 作用域内枚举

传统的C++枚举提供了一种创建名称常量的方式，但其类型检查相当低级。另外，枚举名的作用域为枚举定义所属的作用域，这意味着如果在同一个作用域内定义两个枚举，他们的枚举成员不能同名。最后，枚举可能不是可完全移植的，因为不同的实现可能选择不同的底层类型。为解决这些问题，C++11 新增了一种枚举。这种枚举使用 class 或 struct 定义：

```c++
enum Old {yes, no, maybe};    // old form
enum class New1 {never, sometimes, often, always}; // new form
enum struct New2 {never, lever, sever}; // new form
```

新枚举要求进行显示限定，以免发生名称冲突。因此，引用特定枚举时，需要使用 New1::never 和 New2::never 等。

### 1.7 对类的修改

#### 1.7.1 显示转换运算符

传统C++的关键字 **explicit** 禁止单参数构造函数导致的自动转换：

```c++
class Plebe {
    Plebe(int); // automatic int-to-plebe conversion
    explicit Plebe(double); // requires explicit use
    // ...
};
Plebe a, b;
a = 5;    // implicit conversion, call Plebe(5)
b = 0.5;  // not allowed
b = Plebe(0.5); // explicit conversion
```

C++11扩展了 **explicit** 的这种用法，使得可对转换函数做类似的处理：

```c++
class Plebe {
    // ...
    // conversion functions
    operator int() const;
    explicit operator double() const;
    // ...
};
Plebe a, b;
int n = a;    // int-to-Plebe automatic conversion
double x = b;  // not allowed
x = double(0.5); // explicit conversion, allowed
```

#### 1.7.2 类内成员初始化

传统C++不支持在类定义中初始化成员，C++11可以这么做：

```c++
class Session {
    int mem1 = 10;
    double mem2 {1966.54};
    // ...
};
```

### 1.8 模板和STL方面的修改

#### 1.8.1 基于范围的for循环

对于内置数组以及包含方法 begin() 和 end() 的类和 STL 容器，可以使用如下的方式进行循环工作：

```c++
int arr[5] = {1, 2, 3, 4, 5};
for (int x : arr)
std::cout << x << std::endl;
// 可使用auto进行类型推断
for (auto x : arr)
std::cout << x << std::endl;
// 可使用引用类型进行修改元素
for (auto & x : arr)
x = std::rand();
```

#### 1.8.2 新的STL容器

C++11新增了STL容器： **forward_list、unordered_map、unordered_multimap、unordered_set** 和 **unordered_multiset**。
C++11还新增了模板 **array**，该模板相对于数组，新增了begin() 和 end() 方法等。

#### 1.8.3 新的STL方法

C++11 新增了STL方法 cbegin()、cend()、crbegin() 和 crend()。是begin()、 end()、rbegin() 和 rend() 方法的const版本。

#### 1.8.4 valarray升级

C++11添加了两个函数（begin() 和 end()），它们都接受valarray作为参数，并返回迭代器。这使得能够将基于范围的STL算法用于 valarray。

#### 1.8.5 摒弃export

#### 1.8.6 尖括号

为避免与运算符 >> 混淆，C++要求在声明嵌套模板时使用空格将尖括号分开：

```c++
std::vector<std::list<int> > vl;
```

C++11 不再这样要求：

```c++
std::vector<std::list<int>> vl;
```

### 1.9 右值引用

传统的 C++ 引用（现在称为左值引用）使得标识符关联到左值。左值是一个表示数据的表达式（如变量名或解除引用的指针），程序可获取其地址。最初，左值可出现在赋值语句的左边，但修饰符 const 的出现使得可以声明这样的标识符，即不能给它赋值，但可获取其地址：

```c++
int n;
int * pt = new int;
const int b = 101;   // can't assign to b, but &b is valid
int & rn = n;        // n identifies datum at address &n
int & rt = *pt;      // *pt identifies datum at address pt
const int & rb = b;  // b identifies const datum at address &b
```

C++11 新增了右值引用，这是使用&&表示的。右值引用可关联到右值，即可出现在赋值表达式右边，但不能对其应用地址运算符的值。右值包括字面常量、诸如x+y等表达式以及返回值的函数（条件是该函数返回的不是引用）：

```c++
int x = 10;
int y = 23;
int && r1 = 13;
int && r2 = x + y;
double && r3 = std::sqrt(2.0);
```

注意，r2关联到的是当时计算 x + y 得到的结果。也就是说，r2关联到的是33，即使以后修改了x 或 y，也不影响到r2。
有趣的是，将右值关联到右值引用导致该右值被存储到特定的位置，且可以获取该位置的地址。也就是说，虽然不能将运算符&用于13，但可将其用于r1。通过将数据与特定的地址关联，使得可以通过右值引用来访问该数据。

```c++
#include <iostream>
inline double f(double tf) { return 5.0*(tf-32.0)/9.0; };
int main() {
    using namespace std;
    double tc = 21.5;
    double && rd1 = 7.07;
    double && rd2 = 1.8 * tc + 32.0;
    double && rd3 = f(rd2);
    cout << " tc value and address: " << tc << ", " << &tc << endl;
    cout << "rd1 value and address: " << rd1 << ", " << &rd1 << endl;
    cout << "rd2 value and address: " << rd2 << ", " << &rd2 << endl;
    cout << "rd3 value and address: " << rd3 << ", " << &rd3 << endl;
    cin.get();
    return 0;
}

// 程序输出如下：
tc value and address: 21.5, 0x7ffeb988e850
rd1 value and address: 7.07, 0x7ffeb988e858
rd2 value and address: 70.7, 0x7ffeb988e860
rd3 value and address: 21.5, 0x7ffeb988e868
```

引入右值引用的主要目的之一是实现移动语义。

## 2. 移动语义和右值引用

### 2.1 为何需要移动语义

定义并实现了一个 MyString 字符串类，该类内部管理一个 char * 数组。这个时候一般都需要实现拷贝构造函数和拷贝赋值函数，因为默认的拷贝是浅拷贝，而指针这种资源不能共享，不然一个析构了，另一个也就完蛋了。

```c++
#include <iostream>
#include <cstring>
#include <vector>
using namespace std;

class MyString
{
public:
    static size_t CCtor; //统计调用拷贝构造函数的次数
public:
    // 构造函数
    MyString(const char* cstr=0){
        if (cstr) {
            m_data = new char[strlen(cstr)+1];
            strcpy(m_data, cstr);
        }
        else {
            m_data = new char[1];
            *m_data = '\0';
        }
}
    // 拷贝构造函数
    MyString(const MyString& str) {
        CCtor ++;
        m_data = new char[ strlen(str.m_data) + 1 ];
        strcpy(m_data, str.m_data);
    }
    // 拷贝赋值函数 =号重载
    MyString& operator=(const MyString& str){
        if (this == &str) // 避免自我赋值!!
            return *this;
        delete[] m_data;
        m_data = new char[ strlen(str.m_data) + 1 ];
        strcpy(m_data, str.m_data);
        return *this;
    }
    ~MyString() {
        delete[] m_data;
    }
    char* get_c_str() const { return m_data; }
private:
    char* m_data;
};
size_t MyString::CCtor = 0;

int main(int argc, char* argv[])
{
    vector<MyString> vecStr;
    vecStr.reserve(1000); //先分配好1000个空间，不这么做，调用的次数可能远大于1000
    for(int i=0;i<1000;i++){
        vecStr.push_back(MyString("hello"));
    }
    cout << "CCtor = " << MyString::CCtor << endl; // output: CCtor = 1000
    return 0;
}
```

以上代码调用了1000次拷贝构造函数，如果 MyString("hello") 构造出来的字符串本来就很长，构造一遍就很耗时了，最后却还要拷贝一遍，而 MyString("hello") 只是临时对象，拷贝完就没什么用了，这就造成了没有意义的资源申请和释放操作，如果能够直接使用临时对象已经申请的资源，既能节省资源，又能节省资源申请和释放的时间。而 C++11 新增加的移动语义就能够做到这一点。移动语义实际上避免了移动原始数据，而只是修改了记录。

### 2.2 一个移动示例

下面通过一个示例演示移动语义和右值引用的工作原理。要实现移动语义就必须增加两个函数：移动构造函数和移动赋值构造函数。

```c++
#include <iostream>
#include <cstring>
#include <vector>
using namespace std;

class MyString
{
public:
    static size_t CCtor; //统计调用拷贝构造函数的次数
    static size_t MCtor; //统计调用移动构造函数的次数
    static size_t CAsgn; //统计调用拷贝赋值函数的次数
    static size_t MAsgn; //统计调用移动赋值函数的次数
public:
// 构造函数
    MyString(const char* cstr=0) {
        if (cstr) {
            m_data = new char[strlen(cstr)+1];
            strcpy(m_data, cstr);
        }
        else {
            m_data = new char[1];
            *m_data = '\0';
        }
    }
    // 拷贝构造函数
    MyString(const MyString& str) {
        CCtor ++;
        m_data = new char[ strlen(str.m_data) + 1 ];
        strcpy(m_data, str.m_data);
    }
    // 移动构造函数
    MyString(MyString&& str) noexcept
        :m_data(str.m_data) {
        MCtor ++;
        str.m_data = nullptr; //不再指向之前的资源了
    }
    // 拷贝赋值函数 =号重载
    MyString& operator=(const MyString& str){
        CAsgn ++;
        if (this == &str) // 避免自我赋值!!
            return *this;

        delete[] m_data;
        m_data = new char[ strlen(str.m_data) + 1 ];
        strcpy(m_data, str.m_data);
        return *this;
    }
    // 移动赋值函数 =号重载
    MyString& operator=(MyString&& str) noexcept{
        MAsgn ++;
        if (this == &str) // 避免自我赋值!!
            return *this;

        delete[] m_data;
        m_data = str.m_data;
        str.m_data = nullptr; //不再指向之前的资源了
        return *this;
    }
    ~MyString() {
        delete[] m_data;
    }
    char* get_c_str() const { return m_data; }
private:
    char* m_data;
};
size_t MyString::CCtor = 0;
size_t MyString::MCtor = 0;
size_t MyString::CAsgn = 0;
size_t MyString::MAsgn = 0;

int main(int argc, char* argv[])
{
    vector<MyString> vecStr;
    vecStr.reserve(1000); //先分配好1000个空间
    for(int i=0;i<1000;i++){
        vecStr.push_back(MyString("hello"));
    }
    cout << "CCtor = " << MyString::CCtor << endl;
    cout << "MCtor = " << MyString::MCtor << endl;
    cout << "CAsgn = " << MyString::CAsgn << endl;
    cout << "MAsgn = " << MyString::MAsgn << endl;
    return 0;
}

/* output：
* CCtor = 0
* MCtor = 1000
* CAsgn = 0
* MAsgn = 0 */
```

可以看到，移动构造函数与拷贝构造函数的区别是，拷贝构造的参数是 const MyString& str，是常量左值引用，而移动构造的参数是 MyString&& str，是右值引用，而 MyString("hello") 是个临时对象，是个右值，优先进入移动构造函数而不是拷贝构造函数。而移动构造函数与拷贝构造不同，它并不是重新分配一块新的空间，将要拷贝的对象复制过来，而是"偷"了过来，将自己的指针指向别人的资源，然后将别人的指针修改为 nullptr，这一步很重要，如果不将别人的指针修改为空，那么临时对象析构的时候就会释放掉这个资源，"偷"也白偷了。

### 2.3 强制移动

对于一个左值，肯定是调用拷贝构造函数了，但是有些左值是局部变量，生命周期也很短，能不能也移动而不是拷贝呢？ C++11 为了解决这个问题，在头文件 utility.h 中提供了 std::move() 方法来将左值转换为右值，从而方便应用移动语义。

```c++
int main() {
    vector<MyString> vecStr;
    vecStr.reserve(1000); //先分配好1000个空间
    for (int i = 0; i < 1000; i++) {
        MyString tmp("hello");
        vecStr.push_back(tmp); //调用的是拷贝构造函数
    }
    cout << "CCtor = " << MyString::CCtor << endl;
    cout << "MCtor = " << MyString::MCtor << endl;
    cout << "CAsgn = " << MyString::CAsgn << endl;
    cout << "MAsgn = " << MyString::MAsgn << endl;

    cout << endl;
    MyString::CCtor = 0;
    MyString::MCtor = 0;
    MyString::CAsgn = 0;
    MyString::MAsgn = 0;
    vector<MyString> vecStr2;
    vecStr2.reserve(1000); //先分配好1000个空间
    for (int i = 0; i < 1000; i++) {
        MyString tmp("hello");
        vecStr2.push_back(std::move(tmp)); //调用的是移动构造函数
    }
    cout << "CCtor = " << MyString::CCtor << endl;
    cout << "MCtor = " << MyString::MCtor << endl;
    cout << "CAsgn = " << MyString::CAsgn << endl;
    cout << "MAsgn = " << MyString::MAsgn << endl;
}

/* output:
CCtor = 1000
MCtor = 0
CAsgn = 0
MAsgn = 0

CCtor = 0
MCtor = 1000
CAsgn = 0
MAsgn = 0
*/
```

对于大多数程序员来说，右值引用带来的主要好处并非是让他们能够编写使用右值引用的代码，而是能够使用利用右值引用实现移动语义的库代码。例如，STL 类现在都有复制构造函数、移动构造函数、复制赋值运算符和移动复制运算符。

## 3. 新的类功能

### 3.1 特殊的成员函数

在原有的4个特殊成员函数（默认构造函数、复制构造函数、复制赋值运算符和析构函数）的基础上，C++11 新增了两个：移动构造函数和移动赋值运算符。这些成员函数是编译器在各种情况下自动提供的。

### 3.2 默认的方法和禁用的方法

C++11 提供了更好地控制要使用的方法：可使用关键字 default 显示地声明这些方法的默认版本；关键字 delete 可用于禁止编译器使用特定方法。

### 3.3 委托构造函数

如果给类提供了多个构造函数，您可能重复编写相同的代码。也就是说，有些构造函数可能需要包含其他构造函数中已有的代码。为了让编码工作更简单、更可靠，C++11 允许您在一个构造函数的定义中使用另一个构造函数。这被称为委托。

```c++
class Notes {
    int k;
    double x;
    std::string st;
public:
    Notes();
    Notes(int);
    Notes(int, double);
    Notes(int, double, std::string);
};

Notes::Notes(int kk, double xx, std::string stt) : k(kk), x(xx), st(stt) { /*do stuff*/ }
Notes::Notes() : Notes(0, 0.01, "oh") { /* do other stuff*/ }
Notes::Notes(int kk) : Notes(kk, 0.01, "oh") { /* do other stuff*/ }
Notes::Notes(int kk, double xx) : Notes(kk, xx, "oh") { /* do other stuff*/ }
```

使用委托构造函数，会多发生一次构造函数的调用，这将会影响运行效率，好处在于能提高开发效率。

## 4. Lambda函数

C++11 新增Lambda函数，其格式如下：
>[捕捉列表] (参数) mutable -> 返回值类型 {函数体}

说明：

* []是lambda的引出符，捕捉列表能够捕捉上下文中的变量，来供lambda函数使用：

  * [var] 表示以值传递方式捕捉变量var
  * [=] 表示值传递捕捉所有父作用域变量
  * [&var] 表示以引用传递方式捕捉变量var
  * [&] 表示引用传递捕捉所有父作用域变量
  * [this] 表示值传递方式捕捉当前的this指针
  * 还有一些组合：
  * [=,&a] 表示以引用传递方式捕捉a,值传递方式捕捉其他变量
  * 注意：
  * 捕捉列表不允许变量重复传递，如：[=,a]、[&,&this]，会引起编译时期的错误

* 参数列表与普通函数的参数列表一致。如果不需要传递参数，可以联连同()一同【省略】。
* mutable 可以取消Lambda的常量属性，因为Lambda默认是const属性；multable仅仅是让Lamdba函数体修改值传递的变量，但是修改后并不会影响外部的变量。
* ->返回类型如果是void时，可以连->一起【省略】，如果返回类型很明确，可以省略，让编译器自动推倒类型。
* 函数体和普通函数一样，除了可以使用参数之外，还可以使用捕获的变量。

从C++11开始，Lambda被广泛用在STL中，比如foreach。与函数指针比起来，函数指针有巨大的缺陷：1.函数定义在别处，阅读起来很困难；2.使用函数指针，很可能导致编译器不对其进行inline优化，循环次数太多时，函数指针和Lambda比起来性能差距太大。函数2指针不能应用在一些运行时才能决定的状态，在没有C++11时，只能用仿函数。使得学习STL算法的代价大大降低。

## 5. 包装器

C++ 提供了多个包装器（wrapper，也叫适配器[adapter]）。这些对象用于给其他编程接口提供更一致或更适合的接口。C++11 提供了模板 bind、men_fn 和 reference_wrapper 以及包装器 function。

### 5.1 包装器 function 及模板的低效性

请看以下代码行：

```c++
answer = ef(q);
```

ef 是什么呢？它可以是函数名、函数指针、函数对象或有名称的 Lambda 表达式。所有这些都是可调用的类型（callable type）。鉴于可调用的类型如此丰富，这可能导致模板的效率极低。为明白这一点，来看一个简单的案例。
首先，在头文件中定义一些模板，如下：
**somedef.h**

```c++
#include <iostream>

template <typename T, typename F>
T use_f(T v, F f)
{
    static int count = 0;
    count++;
    std::cout << "  use_f count = " << count
              << ", &count = " << &count << std::endl;
    return f(v);
}

class Fp
{
private:
    double z_;
public:
    Fp(double z = 1.0) : z_(z) {}
    double operator() (double p) { return z_ * p; }
};

class Fq
{
private:
    double z_;
public:
    Fq(double z = 1.0) : z_(z) {}
    double operator() (double p) { return z_ + p; }
};
```

模板 use_f 使用参数 f 表示调用类型：

```c++
return f(v);
```

接下来如下调用模板函数 use_f() 6次。
**callable.cpp**

```c++
#include "somedef.h"
#include <iostream>

double dub(double x) { return 2.0 * x; }
double square(double x) { return x * x; }

int main() {
    using std::cout;
    using std::endl;

    double y = 1.21;
    cout << "Function pointer dub:\n";
    cout << "  " << use_f(y, dub) << endl;
    cout << "Function pointer square:\n";
    cout << "  " << use_f(y, square) << endl;
    cout << "Function pointer Fp:\n";
    cout << "  " << use_f(y, Fp(5.0)) << endl;
    cout << "Function pointer Fq:\n";
    cout << "  " << use_f(y, Fq(5.0)) << endl;
    cout << "Lambda expression 1:\n";
    cout << "  " << use_f(y, [](double u) { return u*u; }) << endl;
    cout << "Lambda expression 2:\n";
    cout << "  " << use_f(y, [](double u) { return u+u/2.0; }) << endl;
    return 0;
}
```

在每次调用中，模板参数 T 都被设置为类型 double。模板参数 F 呢？每次调用时，F 都接受一个 double 值并返回一个 double 值，因此在6次 use_f() 调用中，好像 F 的类型都相同，因此只会实例化模板一次。
但正如下面的输出表明的，这种想法太天真了：

```text
Function pointer dub:
  use_f count = 1, &count = 0x6018cc
  2.42
Function pointer square:
  use_f count = 2, &count = 0x6018cc
  1.4641
Function pointer Fp:
  use_f count = 1, &count = 0x6018d0
  6.05
Function pointer Fq:
  use_f count = 1, &count = 0x6018d4
  6.21
Lambda expression 1:
  use_f count = 1, &count = 0x6018c4
  1.4641
Lambda expression 2:
  use_f count = 1, &count = 0x6018c8
  1.815
```

模板函数 use_f() 有一个静态成员 count，可根据它的地址确定模板实例化了多少次。**有5个不同的地址，这表明模板 use_f() 有5个不同的实例化**。
为了解其中的原因，请考虑编译器如何判断模板参数 F 的类型。首先，来看下面的调用：

```c++
use_f(y, dub);
```

其中 dub 是一个函数的名称，该函数接受一个 double 参数并返回一个 double 值。函数名是指针，因此参数 F 的类型为 `double(*)(double)`：一个指向这样的函数的指针。
下一个调用如下：

```c++
use_f(y, square);
```

第二个参数的类型也是 `double(*)(double)`，因此该调用使用的 use_f() 实例化与第一个调用相同。
在接下来的 use_f() 调用中，第二个参数为对象，F 的类型分别为 Fp 和 Fq，因为将为这些 F 值实例化 use_f() 模板两次。最后，最后两个调用将 F 的类型设置为编译器为 Lambda 表达式使用的类型。

### 5.2 修复问题

注意在 `callable.cpp` 中的函数指针、函数对象和 Lambda 表达式有一个相同的地方，它们都接受一个 double 参数并返回一个 double 值。可以说它们的调用特征标（call signature）相同，因此这6个实例的调用特征标都是 `double(double)`。
模板 function 它从调用特征标的角度定义了一个对象，可用于包装调用特征标相同的函数指针、函数对象和 Lambda 表达式。例如，下面的声明创建了一个名为 fdci 的 function 对象，它接受一个 char 参数和一个 int 参数，并返回一个 double 值：

```c++
std::function<double(char, int)> fdci;
```

然后，可以将接受一个 char 参数和一个 int 参数，并返回 double 值的任何函数指针、函数对象或 Lambda 表达式赋给它。
如 `callable.cpp` 中，所有可调用参数的调用特征标都相同：`double(double)`。因此，可以使用 `std::function<double(double)>` 创建包装器，以达到 use_f() 只被实例化一次的目的，修改的程序清单如下：
在 somedef.h 中，将模板 use_f() 第二个参数声明为 function 包装器对象，如下所示：

```c++
...
#include <functional> // 包装器 function 是在头文件 functional 中声明的

template <typename T>
T use_f(T v, std::function<T(T)> f)
{
    static int count = 0;
    count++;
    std::cout << "  use_f count = " << count
              << ", &count = " << &count << std::endl;
    return f(v);
}

...
```

修改 callable.cpp 中的调用方式如下：

```c++
#include "somedef.h"
#include <functional> // 包装器 function 是在头文件 functional 中声明的
#include <iostream>

double dub(double x) { return 2.0 * x; }
double square(double x) { return x * x; }

int main() {
    using std::cout;
    using std::endl;
    using std::function;

    double y = 1.21;
    // 调用方式一：
    typedef function<double(double)> fdd;
    cout << "Function pointer dub:\n";
    cout << "  " << use_f(y, fdd(dub)) << endl;
    cout << "Function pointer square:\n";
    cout << "  " << use_f(y, fdd(square)) << endl;
    cout << "Function pointer Fp:\n";
    cout << "  " << use_f(y, fdd(Fp(5.0))) << endl;
    cout << "Function pointer Fq:\n";
    cout << "  " << use_f(y, fdd(Fq(5.0))) << endl;
    cout << "Lambda expression 1:\n";
    cout << "  " << use_f(y, fdd([](double u) { return u*u; })) << endl;
    cout << "Lambda expression 2:\n";
    cout << "  " << use_f(y, fdd([](double u) { return u+u/2.0; })) << endl;
    cout << "\n\n";
    // 调用方式二：
    /*
     * 参数 dub、Fp(5.0) 等本身的类型并不是 function<double(double)>，因此在 use_f 后面使用了 <double>
     * 来指出所需的具体化。这样，T 被设置为 double，而 std::function<T(T)> 变成了 std::function<double(double)>
     */
    cout << "Function pointer dub:\n";
    cout << "  " << use_f<double>(y, dub) << endl;
    cout << "Function pointer square:\n";
    cout << "  " << use_f<double>(y, square) << endl;
    cout << "Function pointer Fp:\n";
    cout << "  " << use_f<double>(y, Fp(5.0)) << endl;
    cout << "Function pointer Fq:\n";
    cout << "  " << use_f<double>(y, Fq(5.0)) << endl;
    cout << "Lambda expression 1:\n";
    cout << "  " << use_f<double>(y, [](double u) { return u*u; }) << endl;
    cout << "Lambda expression 2:\n";
    cout << "  " << use_f<double>(y, [](double u) { return u+u/2.0; }) << endl;
    return 0;
}
```

下面是该程序的输出：

```text
Function pointer dub:
  use_f count = 1, &count = 0x6054c0
  2.42
Function pointer square:
  use_f count = 2, &count = 0x6054c0
  1.4641
Function pointer Fp:
  use_f count = 3, &count = 0x6054c0
  6.05
Function pointer Fq:
  use_f count = 4, &count = 0x6054c0
  6.21
Lambda expression 1:
  use_f count = 5, &count = 0x6054c0
  1.4641
Lambda expression 2:
  use_f count = 6, &count = 0x6054c0
  1.815


Function pointer dub:
  use_f count = 7, &count = 0x6054c0
  2.42
Function pointer square:
  use_f count = 8, &count = 0x6054c0
  1.4641
Function pointer Fp:
  use_f count = 9, &count = 0x6054c0
  6.05
Function pointer Fq:
  use_f count = 10, &count = 0x6054c0
  6.21
Lambda expression 1:
  use_f count = 11, &count = 0x6054c0
  1.4641
Lambda expression 2:
  use_f count = 12, &count = 0x6054c0
  1.815
```

从上述输出可知，count 的地址都相同，而 count 的值表明，use_f() 被调用了12次，这表明只有一个实例，并调用了该实例12次，这缩小了可执行代码的规模。

### 5.3 bind

很多 STL 算法都使用函数对象 一一 也叫函数符（functor）。函数符是可以以函数方式与（）结合使用的任意对象。这包括函数名、指向函数的指针和重载了()运算符的类对象（即定义了函数 operator()() 的类）。
例如，可以像这样定义一个对象：

```c++
class Linear
{
private:
    double slope;
    double y0;
public:
    Linear(double sl_ = 1, double y_ = 0)
        : slope(sl_), y0(y_) {}
    double operator()(double x) { return y0 + slope * x; }
};
```

这样，重载的()运算符将使得能够像函数那样使用 Linear 对象：

```c++
Linear f1;
Linear f2(2.5, 10.0);
double y1 = f1(12.5);
double y2 = f2(0.4);
```

函数符的概念如下：

* 生成器（generator）是不用参数就可以调用的函数符。
* 一元函数（unary function）是用一个参数就可以调用的函数符。
* 二元函数（binary function）使用两个参数可以调用的函数符。

在 C++98 中有两个函数 bind1st 和 bind2nd，它们将二元函数转换为一元函数，区别就是 bind1st 用于绑定第一个参数，bind2nd 用于绑定第二个参数，都只能绑定一个参数。
C++98 提供的这些特性已经由于 C++11 的到来而过时，由于各种限制，我们经常使用 bind 而非 bind1st 和 bind2nd。
**过时的 bind1st 和 bind2nd 的用法：**

```c++
vector<int> coll {1, 2, 3, 4, 5, 11, 22, 5, 12};
// 查找元素值大于10的个数
int count = std::count_if(coll.begin(), coll.end(),              // 范围
                          std::bind1st(std::less<int>(), 10));   // 将10绑定到 std::less 的第一个参数，也就是10小于......
// 查找第一个元素值大于10的元素
std::find_if(coll.begin(), coll.end(),              // 范围
             std::bind2nd(std::greater<int>(), 10); // 将10绑定到 std::greater 的第二个参数，也就是......大于10
```

**C++11 中的 bind 的用法：**

```c++
// 计算 x + 10 的结果：
// function object 内部调用 plus<>（也就是 operator+），以占位符 placeholders_1 为第一个参数，
// 以10位第二个参数，占位符_1表示实际传入此表达式的第一实参，返回“实参+10”的结果值
auto plus10 = std::bind(std::plus<int>(), std::placeholders::_1, 10);
std::cout << plus10(7) << std::endl; // 输出 17

// 计算 (x + 10) * 2 的结果：
auto mul2 = std::bind(std::multiplies<int>(),
                     std::bind(std::plus<int>(), std::placeholders::_1, 10),
                     2);
std::cout << mul2(7) << std::endl; // 输出 289
```

注意，上面使用的 less&lt;int&gt;(), gearter&lt;int&gt;(), plus&lt;int&gt;() 以及 multiplies&lt;int&gt;() 等都是 C++ 预定义的 functor。因此我们可以知道，bind 可以把参数绑定到函数对象上。
C++98 的 bind1st 和 bind2nd 局限在于只能绑定一个参数，而 std::bind 可以绑定任意多个参数，使用起来更加方便，如下：

```c++
#include <iostream>
#include <functional>

void fn(int n1, int n2, int n3) {
    std::cout << n1 << " " << n2 << " " << n3 << std::endl;
}

int fn2() {
    std::cout << "fn2 has called.\n";
    return -1;
}

int main()
{
    using namespace std::placeholders;
    auto bind_test1 = std::bind(fn, 1, 2, 3);
    auto bind_test2 = std::bind(fn, _1, _2, _3);
    auto bind_test3 = std::bind(fn, 0, _1, _2);
    auto bind_test4 = std::bind(fn, _2, 0, _1);

    bind_test1();//输出1 2 3
    bind_test2(3, 8, 24);//输出3 8 24
    bind_test2(1, 2, 3, 4, 5);//输出1 2 3，4和5会被丢弃
    bind_test3(10, 24);//输出0 10 24
    bind_test3(10, fn2());//输出0 10 -1
    bind_test3(10, 24, fn2());//输出0 10 24，fn2会被调用，但其返回值会被丢弃
    bind_test4(10, 24);//输出24 0 10
    return 0;
}
```

除此之外，bind 还可以把参数绑定到普通函数、类成员函数、甚至数据成员等。

**std::bind 绑定普通函数：**

```c++
double Divide(double x, double y) { return x/y; }
auto fn = std::bind(&Divide, std::placeholders::_1, 2); // 绑定第二个参数
std::cout << fn(10) << std::endl; // 输出5
```

**std::bind 绑定一个成员函数：**

```c++
struct Foo {
    void print_sum(int n1, int n2)
    {
        std::cout << n1 + n2 << '\n';
    }
    int data = 10;
};
int main()
{
    Foo foo;
    auto f = std::bind(&Foo::print_sum, &foo, 95, std::placeholders::_1);
    f(5); // 输出100
}
```

**std::bind 绑定一个引用参数：**

```c++
#include <iostream>
#include <functional>
#include <vector>
#include <algorithm>
#include <sstream>

ostream & print(ostream &os, const string& s, char c)
{
    os << s << c;
    return os;
}

int main()
{
    vector<string> words{"helo", "world", "this", "is", "C++11"};
    char c = ' ';
    ostringstream os;
    // ostream不能拷贝，若希望传递给bind一个对象，
    // 而不拷贝它，就必须使用标准库提供的ref函数
    for_each(words.begin(), words.end(),
             bind(print, std::ref(os), std::placeholders::_1, c)); // 此处可以不用显示的指定&print，普通函数做实参时，会隐式转换为函数指针
    cout << os.str() << endl;
}
```

**使用 bind 要注意的地方：**

* bind 预先绑定的参数需要传具体的变量或值进去，对于预先绑定的参数，是 `pass-by-value` 的。除非该参数被 std::ref 或者 std::cref 包装，才 `pass-by-reference`。
* 对于不事先绑定的参数，需要传 std::placeholders 进去，从_1开始，依次递增。placeholder 是 `pass-by-reference` 的。
* bind 的返回值是可调用实体，可以直接赋给 std::function 对象。
* 对于绑定的指针、引用类型的参数，使用者需要保证在可调用实体调用之前，这些参数是可用的。
* 类的this可以通过对象或者指针来绑定。
