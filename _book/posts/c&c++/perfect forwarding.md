#### **lvalue and rvalue**

左值和右值是表达式值的两种属性，左值，lvalue,(located value),代表这个值有确定的内存位置，可对其取地址。右值通常表示这个值是临时的，生命期很短，不能对其取地址，字面常量，返回值的函数，临时对象等都是右值。

左值引用 T&,可以引用左值，不能引用右值。 右值引用T&& 只能引用右值，不能引用左值。但是`const T&` 可以引用右值。

在C++11引入右值引用之前，右值只能存在于右值表达式中，引入右值引用之后，右值的生命期和右值引用关联，即使表达式已经计算完了，其结果可以依然一直存在直到其右值引用的生命期结束。比如我们可以写如下代码
```c++
string Hello()
{
    Return “Hello World”;
}
string &&rvalRef = Hello();
rvalRef = “Hello”;
std::cout << rvalRef << std::endl; // Outputs “Hello”
```

同时，在c++11之前，可以使用const 左值引用引用右值，但是这样不可更改右值，现在，有了右值引用，右值也可以被更改了。

需要注意的是，右值引用本身是左值，也就是上述代码中`string &&rvalRef = Hello();`,rvalRef本身是左值

#### **引用折叠**
引用折叠通常被用在模板实例化过程中，具体折叠规则如下
* T& & --> T&
* T& && --> T&
* T&& & --> T&
* T&& && --> T&&

#### **模板参数类型推导
在实例化函数模板的时候，模板类型不需要程序员显示提供，编译器可以自行推导

通常一个函数模板的形式如下
```
template<typename T>
void f(ParamType param);
```
其中ParamType可能为T, T&, const T&, T&& , T*, const T *



* ParamType 为T
这种情况也就是所谓的按值传递。这就意味着param就是完全传给他的参数的一份拷贝——一个完全新的对象。基于这个事实可以从实参给出推导的法则：
1. 如果实参不是指针，T和ParamType的类型都是非引用类型，cv限定符去除。比如实参是int、int&、int&&、const int、const int&，只要不是指针，T和ParamType最终都是int；
2. 如果是指针，T和ParamType都是对应的指针类型，cv限定符保留；但是指针本身的cv属性不会保留
```c++
#include <iostream>

template<typename T>
void f(T param) {}            // param现在是pass-by-value

int main(){
    int x = 27; 
    const int cx = x;
    const int& rx = x; 
    const char* const ptr = "Fun with pointers";

    f(x);                       // T和param的类型都是int
    f(cx);                      // T和param的类型也都是int
    f(rx);                      // T和param的类型还都是int
    f(ptr);                      // T和param的类型是const char*
    f(27);                     //T和param的类型还都是int,
}


```

* ParamType 为T&
对于T&，不管表达式是什么，param一定是左值引用类型（表达式是指针则param是指针的引用），cv限定符保留；

```c++
#include <iostream>

template<typename T>
void f1(T& param) {}

void test_f1() {
    int i = 10;
    int& r = i;
    int&& rr = 10;
    const int& cr = i;
    int* p = &i;
    const int* const q = p;

    f1(i);  // param是int& => T&是int& => T是int
    f1(r);  // param是int& => T&是int& => T是int
    f1(rr); // param是int& => T&是int& => T是int
    f1(cr); // param是const int& => T&是const int& => T是const int
    f1(p);  // param是int* & => T&是int* & => T是int*
    f1(q);  // param是const int* const & => T&是const int* const & => T是const int* const
    f1(27); //param必须是左值引用类型，左值引用无法引用右值，报错
}

```

* ParamType 为T* 
和T&类似，param一定是指针类型，cv限定符保留
```c++
#include <iostream>
template<typename T>
void f2(T* param) {}

void test_f2() {
    int i = 10;
    int& r = i;
    const int& cr = i;
    int* p = &i;
    const int* q = p;

    f2(&i); // param是int* => T*是int* => T是int
    f2(&cr); // param是const int* => T*是const int* => T是const int
    f2(p); // param是int* => T*是int* => T是int
    f2(q); // param是const int* => T*是const int* => T是const int
}

```


* ParamType 为const T&
param为const 引用
```c++
template<typename T>
void f4(const T& param){}

void test_f4() {
    int x = 27;  //x是左值
    const int cx = x;  //cx是左值
    const int& rx = x;  //rx是左值

    f4(x);     // param的类型是const int&，T是int
    f4(cx);    // param的类型是const int&,T是int
    f4(rx);    // param的类型是const int&,T是int
}


```

* ParamType 为T&& 
如果实参是左值，假设类型为U,则param是左值引用U&，cv限定符保留，即采用和T&相同的规则，但是从T&&转换到U&, 需要应用引用折叠，T的类型为U&。
如果实参是右值，假设类型为U，则param是右值引用类型U&&,T为U
```c++
#include <iostream>

template<typename T>
void f(T&& param) {}  // param现在是一个通用的引用

int main() {
    int x = 27; 
    const int cx = x; 
    const int& rx = x; 

    f(x);  // x是左值，所以param是int&, T的类型是int&,int& &&->int&
    f(cx);  // cx是const左值，所以param是const int&, T的类型是const int&,const int& &&-> const int& 
    f(rx);  // rx是const左值，所以param是const int&, T的类型是const int&,const int& &&-> const int& 
    f(27);  // 27是右值，所以param是int&&,T的类型是int
}

```

* 数组 和 指向数组的指针
数组类型和指针指向一个数组这是两个不同的类型，如果ParamType是T，那么数组会被推导成指针类型，如果ParamType是T&，那么数组就会被推导成数组类型，可以使用sizeof求数组的大小
```c++
#include <iostream>

template<typename T>
void f(T param) {}   // param现在是pass-by-value

int main(){
    // name的类型是const char[13]
    const char name[] = "J. P. Briggs";     
    f(name);  // param类型是char char*,T是const char* 
    const char * ptrToName = name;  // 指向数组的指针
    f(ptrToName);  // param类型是char char*，T是const char* 
}
如果ParamType是T&
```c++
#include <iostream>

template<typename T>
void f(T& param) {} 

int main(){
    // name的类型是const char[13]
    const char name[] = "J. P. Briggs";     
    f(name);  // param类型是char char(&)[13],T是const char[13]

    const char * ptrToName = name;          // 数组被退化成指针
    f(ptrToName);  // param类型是char char* &，T是const char* 
}

```

* 函数退化成指针
  函数作为参数会被退化成函数指针，如果参数类型是T，函数会被推导成一个函数指针，指向这个函数，如果参数类型是T&，那么函数会被推导成一个指向函数的引用。
```c++
#include <iostream>

// someFunc是一个函数,类型是void(int, double)
void someFunc(int, double) {}    

template<typename T>
void f1(T param) {} 

template<typename T>
void f2(T& param) {}  


int main(){
    f1(someFunc);   // param被推导成函数指针, 类型是void(*)(int, double)
    f2(someFunc);   // param被推导成指向函数的引用, 类型时void(&)(int, double)
}
```

其实在汇编代码中可以直接看到函数模板参数推导的结果
```
push    rbp
mov     rbp, rsp
mov     edi, OFFSET FLAT:someFunc(int, double)
call    void f1<void (*)(int, double)>(void (*)(int, double)) ;//推导成函数指针
mov     edi, OFFSET FLAT:someFunc(int, double)
call    void f2<void (int, double)>(void (&)(int, double)) ;//指向函数的引用
mov     eax, 0
pop     rbp
ret

```

#### **Perfect forwarding**

有了以上基础，可以完全理解perfect forwarding技术了。


Perfect forwarding is a technique in C++ where a function template can pass its arguments to another function, while preserving the type, value category (lvalue, rvalue), and const-volatile qualifiers of the arguments. This enables the called function to behave as if it received the arguments directly from the original caller, resulting in optimal performance and proper execution.

```c++
#include <utility>

template <typename Func, typename... Args>
auto wrapper(Func&& func, Args&&... args) -> decltype(std::forward<Func>(func)(std::forward<Args>(args)...))
{
    return std::forward<Func>(func)(std::forward<Args>(args)...);
}

void foo(int& x) { /*...*/ }
void foo(const int& x) { /*...*/ }
void foo(int&& x) { /*...*/ }

int main()
{
    int a = 0;
    const int b = 1;
    wrapper(foo, a); // Calls foo(int&)
    wrapper(foo, b); // Calls foo(const int&)
    wrapper(foo, 2); // Calls foo(int&&)
}
```

if without std::forward
```c++
#include <iostream>
#include <utility>

void process(int& x) {
    std::cout << \"Lvalue reference" << std::endl;
}

void process(int&& x) {
    std::cout << "Rvalue reference" << std::endl;
}

template<typename T>
void call_process(T&& arg) {
    process(arg); // Removed std::forward
}

int main() {
    int x = 5;
    
    call_process(x); // Lvalue reference
    call_process(42); // Rvalue reference

    return 0;
}
```

call_process(42) will call to  process(int& x), Even if arg is an rvalue reference , it has a name, so it is treated as an lvalue inside the function. Therefore, the process(int& x) overload will be called.


<https://eli.thegreenplace.net/2014/perfect-forwarding-and-universal-references-in-c/>

