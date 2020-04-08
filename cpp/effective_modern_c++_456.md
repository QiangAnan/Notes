

## 第五章：右值引用、move语义、完美转发
- move语义：作用是将昂贵的拷贝操作替换为代价较小的move移动操作。
move语义使得move-only类型的创建成为可能，比如unique_ptr，future, thread...
- 完美转发：能够接受任意参数的函数模板，并将之转发到其他函数。
- 定律：函数的参数永远是一个左值，即使声明的类型是右值引用如 `void func(Widget &&w);`, 判断变量是左值的简单方式是对其取地址。

### 条款23：理解std::move和std::forward
move和forward只是两个执行转换的模板函数， `move无条件的将实参转换为一个右值`，`forward在条件满足是将参数转换为右值`。
```cpp
// c++11 move实现
template<typename T>
typename remove_reference<T>::type&& move(T &&param) 
{
    using returnType = typename remove_reference<T>::type&&;
    return static_cast<returnType>(param);
}
// c++14 move实现
template<typename T>
decltype(auto) move(T &&param)
{
    using returnType = remove_reference_t<T>&&;
    return static_cast<returnType>(param);
}

// c++11 forward实现版本
template<typename T>  
T&& forward(typename remove_reference<T>::type& param) 
{
    return static_cast<T&&>(param); // T必须为非引用类型才能返回右值
}
template<typename T>  
T&& forward(typename remove_reference<T>::type&& param)  // 右值版本
{
    return static_cast<T&&>(param); // T必须为非引用类型才能返回右值
}

// c++14 forward实现版本
template<typename T>
T&& forward(remove_reference_t<T> &param) 
{
    return static_cast<T&&>(param); // T必须为非引用类型才能返回右值
}

// forward通常配合外层函数使用，外层还是是个万能模板函数，如下，param为右值时，
// 推导T为非引用类型，返回的是右值引用，para为左值时T为左值时，返回左值引用。
// 根据所有函数形参都是左值的定律，可以得出，调用forward的版本一直是左值引用版本，不管param传入的类型是左值还是右值
template<typename T>
void func(T &&param) 
{
    somefunc(std::forward<T>(param)); 
}
```
- param经过move后，返回param的右值类型，如果param被const修饰，返回的右值也有const属性，这时需要看接收移动的变量是否支持const右值，常见是不支持的，所以记住，`需要移动的变量不要声明为const`
```cpp
void func(const string text) 
{
    // 经过move后，返回const右值引用，此时给s赋值，s的构造函数不存在右值const类型，
    // 所以可能会被const &类型的构造接收，从而产生构造行为，如果没有符合的const构造接收，则会报错
    string s(std::move(text));
}
```

### 条款24 区分通用引用与右值引用

- 通用引用： 模板函数`T &&` 或者 `auto &&`, 存在类型推导, 初始化值类型决定了它是左值也还是右值。

如果一个函数模板形参具备 T&&格式且 T 类型需要推导，或者一个对象声明为
auto&&，那么这个形参或对象就是一个通用引用。通用引用如果用右值初始化，就是右值引用，如果用左值初始化，就是左值引用。
```cpp
template<typename T> 
void func(T &&) {};

auto &&var2 = var1; // int举例能推导：int, int&, 具备const和volatile属性
```
- 右值引用： 具体类型`type &&`，不存在类型推导
```cpp
void f(Widget &&w) {};
Widget &&w2 = w1;
```
- 注意：以下推导要视情况而定
```cpp
template<typename T>
void func(std::vector<T> &&param) {}; // param为右值引用，T不作为&&的类型推导。

// const属性会破会通用引用属性
template<typename T>
void func(const T&& param){}; // rvalue

// 类中的函数类型推导也可能不为通用引用
template<class T, class Allocator = allocator<T>>
class vector {
public:
    void push_back(T &&x); // x为rvalue，在类vector实例化时就确定了具体类型
public:
    template<class... Args> 
    void emplace_back(Args&&... args); // args不受T控制，为通用引用
};

// auto &&为通用引用
auto timeFunc = [] (auto &&func, auto &&... params) {
    std::forward<decltype(func)>(func)(
        std::forward<decltype(params)>(params)...
    );
}
```

### 条款25：对右值引用使用std::move, 对通用引用（万能引用）使用std::forward（完美转发)
```cpp
// 右值引用, std::move转移所有权
class Widget {
public:
    Widget(Widget&& rhs) // rhs is rvalue reference
        : name( std::move(rhs.name)),
          p( std::move(rhs.p))
    {}
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

// 通用引用：std::forward转发
class Widget {
public:
    template<typename T>
    void setName(T&& newName) // newName is universal reference
    { 
        // 这里不能用move，move会将传入的左值参数给转移了，导致入参左值失效
        name =  std::forward<T>(newName); 
    }  
};
```
- 参数个数类型无限制的函数模板 使用万能引用+完美转发
```cpp
template<typename T, class... Args>
shared_ptr<T> make_shared(Args&&... args); // c++11  
// std::forward<Args>(args)...
template<typename T, class... Args>
unique_ptr<T> make_unique(Args&&... args); // c++14
```
- 返回值与移动语义
  - 值传递返回且返回值不是局部变量或者引用参数时，move、forward返回值可以避免返回值拷贝到函数返回值位置的拷贝动作
  - RVO：return value optimization返回值优化：以下两个条件满足的情况下，不需要对返回值进行move或者forward，因为编译器会执行去拷贝动作直接移动给返回值（局部变量在返回值内存中构造），或者编译器会隐式的move返回值（返回值为参数，直接move到参数的内存中）。
    - 局部对象与返回值类型相同
    - 返回的就是那个局部对象


```cpp
// 返回为值传递，lhs为右值，可以保存lhs和rhs相加的和。 std::move
Matrix operator+( Matrix&& lhs, const Matrix& rhs) 
{ 
    lhs += rhs;
    // return 语句中，通过将 lhs 转换为右值，lhs 将被移动到函数的返回值位置。
    // 如果 Matrix 不支持移动，则将其转换为一个右值也不会造成伤害，因为右值将被 Matrix
的拷贝构造函数简单地拷贝
    return  std::move(lhs); //  move  lhs into
    // 因为 lhs 是左值，这将迫使编译器将其复制到返回值的位置
    return lhs;
} 

// 返回值传递，参数为通用引用。 std::forward
template<typename T>
Fraction // by-value return
reduceAndCopy(T&& frac) // universal reference param
{
    frac.reduce();
    return  std::forward<T>(frac); // move rvalue into return
} 

// RVO
Widget makeWidget() 
{
    Widget w;
    return w; // w的参数直接构造在返回值内存中，无需拷贝和移动 
}
Widget makeWidget( Widget w) 
{
    return w; // 隐式为std::move
}
```



### 条款26：避免对通用引用重载
通用引用函数是c++中最贪吃的函数，除了完全匹配，否则都会匹配通用引用函数，然而匹配成通用引用函数的实际与预期不一致可能造成意想不到的编译或运行时错误，
因此最好不要重载通用引用，如果确实需要重载，参见下一条款说明。

### 条款27：熟悉对通用引用重载的可选方式
- 为了解决条款26中的问题，可以采用：放弃重载、传const左值引用（精确匹配）、传值（非通用引用）等非常局限性的解决方法（不支持完美转发：传左值转发左值，传右值转发右值）。
- 既能完美转发又能重载：`标签分派 Tag dispatch`
```cpp
template<typename T>
void logAndAdd(T&& name)
{
  logAndAddImpl(std::forward<T>(name), std::is_integral<<typename std::remove_reference<T>::type>()); // 封装一层，通过参数is_integral区分重载
  // c++14 -->  logAndAddImpl(std::forward<T>(name), std::is_integral<<std::remove_reference_t<T>>()); 
}

// 第二个参数可取： std::true_type、 std::false_type

// 重载false
template<typename T> 
void logAndAddImpl(T&& name, std::false_type)  // 参数时false
{ 
    auto now = std::chrono::system_clock::now(); 
    log(now, "logAndAdd"); 
    names.emplace(std::forward<T>(name));
}

// 重载true
std::string nameFromIdx(int idx);
void logAndAddImpl(int idx, std::true_type) // true
{ 
    logAndAdd(nameFromIdx(idx));
} 
```
- std::enable<br>
标签分派虽然能解决通用引用数量较少的参数重载，如果参数较多，总是不能避免覆盖不了的被通用引用匹配上的非预期场景。
为了避免这些场景绕过标签分派，c++使用了杀伤力更大的工具`std::enable_if`, 使用 std::enable_if 的模板只有在满足 std::enable_if 指定的条件时才启用.
std::enable_if的工作原理依靠`SFINAE`技术。
为了更好理解下面例子，我们需要
  - 引入`std:is_same`，`std::is_same<T1, T2>::value`用来判断两个参数是否是同一类型，
  - 引入`std::decay`, 使用`std::decay<T>::type`去除T的const、volatile和引用的限定属性.
  - 引入`std::is_base_of`, `std::is_base_of<T1, T2>::value`用来判断两个类型是否是继承关系
```cpp
// 初识用法
class Person {
public:
  template<typename T, typename = typename std::enable_if<condition>::type>
  explicit Person(T&& n);
};

// typename std::decay<T>::type 去除T 引用、cv属性后的类型； 
// !std::is_same<T1, T2>::value 是否相同的value
// typename std::enable_if<bool>::type 是否满足enable_if条件
class Person {
public:
    template<typename T, 
             typename = typename std::enable_if<
                  !std::is_same<Person, typename std::decay<T>::type>::value // is_same
                >::type>
    explicit Person(T&& n);

    template<typename T, 
             typename = typename std::enable_if<
                  !std::is_base_of<Person, typename std::decay<T>::type>::value // is_base_of 用在子父类继承上
                >::type>
    explicit Person(T&& n);

    // c++14通过对别名type的优化(去掉显示指明类型的typename和后缀::type)，可以简化成
    template<typename T, typename = std::enable_if_t<!std::is_same<Person, std::decay_t<T>>::value>>
    explicit Person(T&& n);
};
```

- 使用enable_if实现标签分类达到的效果
```cpp
class Person {
public:
    // 满足非person类或继承类并且非int类型的参数 的通用引用函数
    template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value &&
                                                     !std::is_integral<std::remove_reference_t<T>>::value>>
    explicit Person(T&& n) 
        : name(std::forward<T>(n)) 
    {
    } 

    explicit Person(int idx) 
    : name(nameFromIdx(idx))
    {
    }

private:
    std::string name;
};
```
- 其他知识点<br>
`is_constructible` 这个类型特性执行编译时检查，以确定是否可以从一个不同类型(或类型集)的对象(或对象集)构造另一个类型的对象，因此断言很容易编写
```cpp
class Person {
public:
    template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value &&
                                                     !std::is_integral<std::remove_reference_t<T>>::value 
                                                    >  
            >
    explicit Person(T&& n)
    : name(std::forward<T>(n))
    {
        // 如果用户代码试图从某个类型创建 Person，而该类型又不能用于构造 std::string，则会生成指定的错误消息。
        static_assert(std::is_constructible<std::string, T>::value, "Parameter n can't be used to construct a std::string");
        ...
    }
};
 
```

