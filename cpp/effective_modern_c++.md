
## 第一章：类型推导 -- **<font color = red>需要多看两边记忆</font>**
```
c++98类型推导：函数模板
c++11：auto、decltype
c++14：decltype(auto)  ^_^
```

### 条款1：理解模板类型推导 
```cpp
template<typename T>
void f(ParamType param);
f(expr);  // 描述expr推导T和ParamType的关系
```
- 模板T是普通引用
```cpp
template<typename T>
void f(T &param) {}
// T可以推导成普通类型和引用类型 int &param/ const int &param(T&可以推导const属性)
```
- 模板T是通用引用
```cpp
template<typename T>
void f(T &&param) {}
// T可以推导成int, int&, const int& ==> int &param, int &&param, const int &param; 具体涉及到万能引用，见条款24
```
- 模板T既不是指针也不是引用: 这就意味着，无论传入的是什么，param 都将生成一个拷贝
```cpp
template<typename T>
void f(T param) {}
// 普通的值传递，不具备&/const的推导，但是可以推导成一个char *,或者const char *；不能为char const *, 因为推导成指针时，param本身不能有const属性。
```
- 数组与指针的转换关系与类型推导
  - 数组引用也可以作为参数
```cpp

const char tt[22] = {0};

template<typename T>
void f1(T para){};
template<typename T>
void f2(T &para){};

f1(tt); // T = const char *, 数组退化成指针
f2(tt); // T = const char(&)[], 数组的引用，并且带const属性

// ***利用数组引用的推导，可以在编译器通过模板推导得出数组的大小***
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T(&)[N]) noexcept { // 返回值是constexpr std::size_t
    return N;// 传入数组，推导出数组引用类型和数组大小
}
// 使用： 
int keyVlaue[] = {2, 3, 4, 5, 6};
arraySize(keyValue); // 推导出T是int，N是5

```
- 函数名与函数指针的转换
```cpp
// 模板函数
template<typename T>
void f(T param) {};
template<typename T>
void f2(T &param) {};
// 函数
void tt(int);
// 函数作为参数传入模板
f(tt); // T推导为 void (*)(int) 函数指针
f2(tt); // param ==> void (&)(int)  函数引用
```

### 条款2：理解auto类型推导
```cpp
// 以下性质和T模板类型推导一致
auto x = 27; // auto -- int
const auto cx = x; // auto -- int 
const auto &rx = x; // auto -- int
auto &&ref1 = x; // auto -- int&
auto &&ref2 = cx;  // auto --const int &  (单独的auto x不会有const属性，auto &&x会有)
auto &&ref3 = 27; // auto -- int

const char name[] = "xxxx";
auto arr1 = name; // auto -- const char *
auto &arr2 = name // auto -- const char (*)[] 数组引用

void func(int);
auto f1 = func; // auto -- void (*)(int)
auto &f2 = func2; // auto -- void (&)(int)

// std::initializer_list<int> 和T有区别
auto x = 23; // auto -- int
auot x2(23); // same

auto x3 = {23}; // auto -- std::initializer_list<int>
auto x4{23}; // same
```
- <font color = red>模板类型推导和auto的区别</font>  -- std::initializer_list<T>的推导<br>
  auto可以推导成initializer_list<T>， 而 T param不行。但是当auto的作用和T作用类似时，不能推导成initializer_list<T>：如下函数返回类型，函数参数、lambda不行。
```cpp
auto func() {
    return {1, 2, 3}; // error auto 出错
}

void func(auto f) {  // error当成模板推导了
    return;
}

auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14…
resetV({ 1, 2, 3 }); // error! can't deduce type
```

## 条款3：理解decltype
- decltype(name): 返回变量name的类型，不增减任何修饰。
- c++14提出 `decltype(auto)`用来解决c++11中显示使用返回值尾部置法。
  - decltype(auto) 可以用来做函数返回值，也可以声明变量
```cpp
// c++11 尾置返回类型，可以使用参数类型推导
template<typename Container, typename Index> 
auto func(Container &c, Index i) ->decltype(c[i]) {
    return c[i];
}

// c++14 decltype(auto)推导返回类型 -- auto作为返回值时是模板推导，引用特性被忽略，值传递
template<typename Container, typename Index>
decltype(auto) func(Container &c, Index i) { 
    return c[i];
}
// 万能引用版本
template<typename Container, typename Index>
auto func(Container &&c, Index i) ->decltype(std::forward<Container>(c)[i]) {
    return std::forward<Container>(c)[i];
}
template<typename Container, typename Index>
decltype(auto) func(Container &&c, Index i) {
    return std::forward<Container>(c)[i];
}

// decltype(auto)
const int &aa = 0;
auto a = aa; // a不具有const和&引用属性
decltype(auto) a2 = aa; // a == const int &
```
- decltype((x)) 给变量x加括号，返回变量x的引用
  - 函数返回时，变量用括号包一下，函数返回类型声明为decltype(auto)来接返回的(x)
```cpp
int x = 0;
decltype((x)) x2 = x;  // x2 == int &

decltype(auto) func() {
    int x = 0;
    return (x);
}
```

### 条款4：了解如何查看推导出的类型
- IDE可以在编写代码阶段查看推导类型
- 编译器在编译的时候可以推导
- 运行时输出 typeid(x).name(), 不同编译器编译出的name不同，如gun编译出i指的是int，pki值得是Point to Const Int

---
## 第二章： auto

### 条款5： 优先使用auto，而非显示的类型声明
- auto x 若x未初始化，会报错，表明必须初始化
- 对于复杂的类型使用auto可以自动推导
- c++11 返回值可以使用auto，c++14时lambda的参数也可以使用auto
```cpp
auto derefUPLess =  // c++11
    [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2) { 
        return *p1 < *p2; 
    }; 
auto derefUPLess =  // c++14
    [](const auto& p1, const auto& p2) { 
        return *p1 < *p2; 
    }; 
```
- auto和function </br>
std::function可以包含任何可调用对象，当接收lambda表达式时，function也能存储闭包。相对于auto接收闭包，function需要定义一个实例对象，对象要占内存，但这个内存并不一定能满足闭包要求的内存，如果超出，需要在堆中申请内存，而auto所占的内存等于闭包的内存。所用通常function的实例对象比auto的变量所占内存更多。另外auto的变量名通常比fucntion短，写法上更简洁。
```cpp
// function：需要声明对象inst, 写法也比较复杂
std::function<bool(const std::unique_ptr<T> &, const std::unique_ptr<T> &)> inst = [](const std::unique_ptr<T> &p1, const std::unique_ptr<T> &p2){ 
    return *p1 < *p2;
};

// auto
auto compare = [](const auto &p1, const auto &p2){
    return *p1 < *p2;
};
```
- 使用auto能避免一些隐式类型的错误
```cpp
// unordered_map的key是const属性的，下面的写法错误
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int> &p : m) {
    // 错误在于map的key为const string
}
// 因为声明的pair<string, int>和实际的<const string, int>不匹配，在for循环时会产生一个临时pair<string, int>用来接收m中的元素强转后结果，p再引用这个临时元素。因为引用的是临时变量，这和预期想要达到的目的不同了。

// 直接使用auto
for (const auto &p : m) {
    // 不会出错，写法也方便
}
```
- 使用auto后，编码的时候类型看起来可能不那么直观，这时要合理借助IDE类型显示，虽然IDE显示不完美，但有总比没有好吧 ^_^

### 条款6 当auto推到的类别不符合要求时，使用带显示类别的初始化物习惯用法
虽然条款5中介绍了auto的好处：避免未初始化变量，减少啰嗦的变量声明，可以直接持有闭包，能够避免一些隐式的类型错误等等； 但是auto不是万能的，在遇到隐形的`代理类`类型时，auto会直接推到成代理类，导致和预期的被代理类类型不同。
```cpp
// 预期场景
std::vector<bool> v(10);
bool test = v[2];
func(test); // void func(bool); 

// 若用auto接收返回值
auto test = v[2]; // test类型为std::vector<bool>::reference
func(test); 
```
这是因为vector<bool>比较特殊，其他vector<T>在使用`operator[]`时返回的都是元素的引用，而vector<bool>返回的是vector<bool>::reference类型对象（这是嵌套在std::vector<bool>里面的类），而使用bool去接收`operator[]`的返回值时会隐式的将代理类vector<bool>::reference强转成bool。 和vector<bool>相同的代理类还有std::bitset的std::bitset::reference<br>

这种情况应该怎么避免呢：强制进行另一次类型转换，这种方法称为带显示类别的初始化物习惯用法
```cpp
auto test = static_cast<bool>(v[2]);
```
---
## 第三章 转向现代c++


--- 

## 第七章：并发API

- c++11提供的并发api是的c++第一次通过标准库实现跨平台。
- 提供的并发操作由 tasks、futures、threads、mutexes、condition、variables、atomic、objects等
  
### 条款35 优先使用task-based编程而非thread-based
