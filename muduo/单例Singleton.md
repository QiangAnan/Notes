



```cpp
namespace muduo
{

namespace detail
{

template<typename T>  // SFINAE
struct has_no_destroy
{
  template <typename C> static char test(decltype(&C::no_destroy)); 
  template <typename C> static int32_t test(...);
  const static bool value = sizeof(test<T>(0)) == 1; // 匹配T::no_destroy是否存在
};

}  // namespace detail

template<typename T>
class Singleton : noncopyable
{
 public:
  Singleton() = delete;
  ~Singleton() = delete;

  static T& instance()
  {
    pthread_once(&ponce_, &Singleton::init); // init只会执行一次
    assert(value_ != NULL);
    return *value_;
  }

 private:
  static void init()
  {
    value_ = new T();
    if (!detail::has_no_destroy<T>::value) // 判断T类型有没有no_destroy函数
    {
      ::atexit(destroy); // 没有no_destroy函数时调用destroy()
    }
  }

  static void destroy()
  {
    // 如果 T为 incompete type, sizeof(T)返回0， 则数组大小为-1，编译报错
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1]; 
    // 消除warning： type T_must_be_complete_type not used
    T_must_be_complete_type dummy; (void) dummy; 

    delete value_;
    value_ = NULL;
  }

 private:
  static pthread_once_t ponce_;
  static T*             value_;
};

// static变量类外初始化
template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;

template<typename T>
T* Singleton<T>::value_ = NULL;

}  // namespace muduo

```


## 1. pthread_once
`int pthread_once(pthread_once_t *once_control, void (*init_routine) (void));` 

本函数使用初值为`PTHREAD_ONCE_INIT`的`once_control`变量保证`init_routine()`函数在本进程执行序列中仅执行一次。

## 2. 模板类中的static
共同的实例模板共享一个static变量<br>
static模板初始化格式:

`template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;`

## 3. complete type 和 incomplete type

incomplete type 定义
```
The following are incomplete types:
• Type void
• Array of unknown size
• Arrays of elements that are of incomplete type
• Structure, union, or enumerations that have no definition
• Pointers to class types that are declared but not defined
• Classes that are declared but not defined
```

Complete类型大小在编译器可以确定，Imcomplete type在编译器不能确定类型大小，`sizeof(imcomplete type) `值为0， 对于`char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1]; `若sizeof(T)返回0， 则数组`T_must_be_complete_type`大小为-1， 编译报错。

[https://blog.csdn.net/q5707802/article/details/79253926?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task](https://blog.csdn.net/q5707802/article/details/79253926?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
[https://blog.csdn.net/encoder1234/article/details/79012336](https://blog.csdn.net/encoder1234/article/details/79012336)
[https://blog.csdn.net/u014618114/article/details/103222250?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task](https://blog.csdn.net/u014618114/article/details/103222250?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
### 4. SFINAE
Substitution failure is not an error,匹配失败并不是错误,意思是用函数模板匹配规则来判断类型的某个属性是否存在,也就是说SFINAE可以作为一种编译期的不完整内省方法.

[https://www.jianshu.com/p/45a2410d4085](https://www.jianshu.com/p/45a2410d4085)
[https://blog.csdn.net/freeelinux/article/details/53428867](https://blog.csdn.net/freeelinux/article/details/53428867)

- SFINAE子父类继承时是否包含某函数的判断
```cpp
// SFINAE失效
#include <iostream>

template<typename T, typename Sig>                                 
struct has_foo {                     
    template <typename U, U> struct type_check;
    template <typename V> static char (& chk(type_check<Sig, &V::foo>*))[1];
    template <typename  > static char (& chk(...))[2]; 
    static bool const value = (sizeof(chk<T>(0)) == 1);
};

struct A {
    void foo();
};

struct B : A {};

int main()
{
    using namespace std;
    cout << boolalpha << has_foo<A, void (A::*)()>::value << endl; // true
    cout << boolalpha << has_foo<B, void (B::*)()>::value << endl; // false
}

// 改进使得SFINAE支持子父类
#include <iostream>

template <typename Type> 
class has_foo
{ 
   class yes { char m;}; 
   class no { yes m[2];}; 
   struct BaseMixin 
   { 
     void foo(){} 
   }; 
   struct Base : public Type, public BaseMixin {}; 

   template <typename T, T t>  
   class Helper{}; 

   template <typename U> 
   static no deduce(U*, Helper<void (BaseMixin::*)(), &U::foo>* = 0);  // 1)

   static yes deduce(...); // 2)
public: 
    // 推导1)，若Type中有foo，则Helper中&U::foo编译不通过(两个基类都有foo)，则应该匹配2)
   static const bool result = sizeof(yes) == sizeof(deduce((Base*)(0))); 
}; 

struct A {
    void foo();
};

struct B : A {};

struct C {};

int main()
{
    using namespace std;
    cout << boolalpha << has_foo<A>::result << endl; // true
    cout << boolalpha << has_foo<B>::result << endl; // true
    cout << boolalpha << has_foo<C>::result; // false
}
```

