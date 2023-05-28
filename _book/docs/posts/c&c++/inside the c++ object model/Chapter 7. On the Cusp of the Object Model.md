**Template**
原本template被视为是对container classes such as Lists and Arrays的一項支持，但现在它已经成为STL的基础，They also are used as idioms for attribute mix-in where, for example, memory allocation strategies ([BOOCH93]) or mutual exclusion mechanisms for synchronizing threads ([SCHMIDT94]) are parameterized.

它甚至被使用于一項所谓的template metaprograms技术： in which class expression templates are evaluated at compile time rather than runtime, thereby providing significant performance improvements 


instantiation(实例化)指的是，将具体的类型绑定到模板参数的过程，比如，有模板函数如下
```c++
template <class Type>
Type min( const Type &t1, const Type &t2 ) { ... }
```
和其使用
```c++
min( 1.0, 2.0 );
```
the instantiation process binds Type to double and creates a program text instance of min() (suitably mangled to give it a unique name within the executable) in which t1 and t2 are of type double.

#### **Template Instantiation**
Consider the following template Point class:
```c++
template <class Type>
class Point
{
public:
  enum Status { unallocated, normalized };
  Point( Type x = 0.0, Type y = 0.0, Type z = 0.0 );
  ~Point();
  void* operator new( size_t );
  void operator delete( void*, size_t );
// ...
private:
  static Point< Type > *freeList;
  static int chunkSize;
  Type _x, _y, _z;
};
```
首先，当编译器看到这个模板类声明时会发生什么？就实际程序而言，什么也不会发生。也就是说，静态数据成员不可用。嵌套的枚举或其枚举值也不可用。

尽管 enum Status 的实际类型在所有 Point 实例化中都是不变的，其枚举值也是如此，但每个枚举值只能通过特定的模板类 Point 实例访问。因此我们可以写成：
```c++
// ok:
Point< float >::Status s;
```
but not
```c++
// error:
Point::Status s;
```
虽然从抽象的角度来看，这两种类型是相同的。（并且，最理想的情况是只生成一个枚举实例。如果无法做到这一点，我们可能需要将枚举因子提取到非模板基类中，以防止多个副本出现。）

同样地， the static data members freeList and chunkSize are not yet available to the program. We cannot write
```c++
// error:
Point::freeList;
```
but must specify the explicit template Point class instantiation with which the freeList member is associated:
```c++
// ok:
Point< float >::freeList;
```
编译器在遇到以上代码时，首先会实例化Point<float> 类，然后生成该类的freeList静态数据成员
```c++
// ok: another instance
Point< double >::freeList;
```
a second freeList instance is generated, this one associated with the double instantiation of the Point class.

如果编译器遇到如下代码呢？会不会具象化一个Point<float>类声明出来？
```c++
Point< float > *ptr = 0;
```

并不会，什么也不会发生。因为指向类对象的指针本身不是一个类对象；编译器不需要知道类的布局或成员。因此，生成Point<float>类是不必要的。

但如果是引用
```c++
const Point< float > &ref = 0;
```
does result in the instantiation of a float instance of Point. 

The actual semantics of this definition expand as follows:
```c++
// internal expansion
Point< float > temporary( float (0) );
const Point< float > &ref = temporary;
```
Why? Because a reference cannot be an alias to "no object." The 0 is treated as an integer value that must be converted into an object of the type
```c++
Point< float >
```
If there is no conversion possible, then the definition is in error and is flagged at compile time.

所以，如果是类对象的定义，不管是由编译器隐式定义的临时对象，还是程序员显示定义的，如下
```c++
const Point< float > origin;
```
会实例化一个Point<float>类，并定义一个origin对象。Point<float>类对象有3个float数据成员。

然而，以上的所有情况，member functions—at least those that are not used，都不会实例化的。Standard C++ requires that member functions be instantiated only if they are used (current implementations do not strictly follow this requirement).  There are two main reasons for the use-directed instantiation rule:

1. Space and time efficiency. If there are a hundred member functions associated with a class, but your program uses only two for one type and five for a second type, then instantiating the additional 193 can be a significant time and space hit.
2. Unimplemented functionality. Not all types with which a template is instantiated support all the operators (such as i/o and the relational operators) required by the complete set of member functions. By instantiating only those member functions actually used, a template is able to support types that otherwise would generate compile-time errors.

以上origin的定义
```c++
const Point< float > origin;
```
只需要实例化Point<float> 的默认构造函数和析构函数。

如下代码
```c++
Point< float > *p = new Point< float >;
```
会实例化Point<float>,operator new， default constructor会被实例化。
(It's interesting to note that although operator new is implicitly a static member of the class and so may not directly access any of its nonstatic members, it is still dependent on the actual template parameter type because its size_t first argument is passed the class size.)

那么函数是何时被实例化的呢？有2中实现策略
* 在编译时。In which case the functions are instantiated within the file in which origin and p are defined.
* 在连接时。In which case the compiler is reinvoked by some auxiliary tool. The template function instances may be placed within this file, some other file, or a separate repository.


#### **Error Reporting within a Template**
假设有如下的模板类声明
```c++
1. template <class T>
2. class Mumble
3. {
4. public$:
5.    Mumble( T t = 1024 )
6.      : _t( t )
7.    {
8.      if ( tt != t )
9.          throw ex ex;
10.   }
11. private:
12.   T tt;
13. }
```

Mumble模板类的声明包含了一系列明显和潜在的错误：
1. 第4行：使用$字符是不正确的。这个错误有两个方面：(1) $不是一个有效的标识符，(2) 在类声明体中只允许public、protected和private标签 (存在$时它不再被识别为关键字public)。
2. 第5行：将t初始化为整数常量1024可能合法也可能不合法，具体取决于T的实际类型。通常情况下，这只能针对每个模板实例进行诊断。
3. 第6行：_t并非成员名称；tt才是。这种类型的错误通常在类型检查阶段发现，during which each name is either bound to a definition or an error is generated.
4. 第8行：不等运算符(!= )是否被定义了取决于T的实际类型。与第二项相同，这只能针对每个模板实例进行诊断。
5. 第9行: 我们意外地输入了两次ex。这是编译过程中解析阶段发现的错误（语言中合法“句子”不能有一个标识符跟随另一个）。
6. 第13行: 我们忘记用分号结束类声明。同样，在编译过程中解析阶段发现了这个错误。

在非模板类声明中，编译器在看到类声明时就会对这六个明显和潜在的错误做针对处理。然而，在模板中并非如此。首先，涉及模板参数的所有类型相关检查都必须推迟到实例化发生时。也就是说，示例中第5行和第8行（上面的2和4项）的潜在错误将针对每个实例进行检查和报告，并且将按类型逐个解决。因此，对于
```c++
Mumble< int > mi;
```
第5行和第8行的代码是正确的。

但是如果是
```c++
Mumble< int* > pmi;
```
ne 8 is correct, but line 5 is a type error—you cannot assign a pointer an integer constant (other than 0).

With the declaration
```c++
class SmallInt
{
public:
  SmallInt( int );
  // ...
};
```
within which the not-equal operator is not defined, the instance
```c++
Mumble< SmallInt > smi;
```
generates an error for line 8, while line 5 is correct. Of course,
```c++
Mumble< SmallInt* > psmi;
```
once again reverses that: Line 8 is again correct, but line 5 is again in error.

那么，在处理模板声明时会出现哪些错误呢？这在一定程度上取决于模板的处理策略。在最初的cfront实现中，模板被完全解析但未进行类型检查。类型检查仅应用于每个特定实例化。因此，在解析策略下，所有词法分析和语法分析错误都将在处理模板声明期间被标记出来。

词法分析器会捕获第4行的非法字符。 The parser itself would likely flag the
```c++
public$: // caught
```
as an illegal label within the class declaration

第6行的错误不会被标记出来,这种错误在类型检查阶段被发现，对于模板类，类型检查被延迟到实例化期间进行
```c++
_t( t ) // not caught
```

It would catch the presence of ex twice in the throw expression of line 9 and the missing semicolon on line 13.

另一种实现策略，模板声明被收集为一系列的lexical tokens。对这些token的解析被延迟到实例化期间。 在实例化时，每个tokens被push到解析器中解析，并应用类型检查，等等操作。
What errors does a lexical tokenizing of the template declaration flag? Very few, in fact; only the illegal character used in line 4. The rest of the template declaration breaks up into a legal collection of tokens.

非成员和成员模板函数在实例化之前也没有完全进行类型检查。在当前的实现中，这会导致一些明显不正确的模板声明编译时没有错误。For example, given the following template declaration of Foo:
```c++
template < class type >
class Foo {
public:
  Foo();
  type val();
  void val( type v );
private:
  type _val;
};
```
cfront, the Sun compiler, and Borland all compile this without complaint:
```c++
// current implementations do not flag this definition
// syntactically legal; semantically in error:
// (a) bogus_member not a member function of class Foo
// (b) dbx not a data member of class
template < class type >
double Foo< type >::bogus_member() { return this->dbx; }
```

再次强调，这是编译器的实现决策。模板机制本身并不排除对模板声明的非类型相关部分进行更严格的错误检查。此类错误可以被发现和标记。但目前为止，没有编译器这样做。

#### **Name Resolution within a Template**

There is a distinction between the program site at which a template is defined (called in the Standard the scope of the template definition) and the program site at which a template is actually instantiated (called the scope of the template instantiation). 

An example of the first is
```c++
// scope of the template definition
extern double foo ( double );
template < class type >
class ScopeRules
{
public:
  void invariant() {
    _member = foo( _val );
  }
  type type_dependent() {
    return foo( _member );
  }
// ...
private:
  int _val;
  type _member;
};
```
and an example of the second is
```c++
//scope of the template instantiation
extern int foo( int );
// ...
ScopeRules< int > sr0;
```
在上述的两个scope中，有2个foo的调用，其中在第一个scope中，one declaration of foo()，`extern double foo ( double );`,而在第二个scope中，however, two declarations are in scope. 

如果我们有如下的调用
```c++
// scope of the template instantiation
sr0.invariant();
```
the question is, which instance of foo() is invoked for the call:
```c++
// which instance of foo()?
_member = foo( _val )
```
The two instances in scope at this point in the program are
```c++
// scope of the template declaration
extern double foo ( double );
// scope of the template instantiation
extern int foo( int );
```
and the type of _val is int.

So which do you think is chosen?  Obviously, the instance chosen is the nonintuitive one:
```c++
// scope of the template declaration
extern double foo ( double );
```
编译器在解析模板中的nonmember name时，具体解析到哪个定义，取决于 whether the use
of the name is dependent on the parameter types used to instantiate the template. 如果对这个name的使用，跟模板实例化的类型参数无关，则使用template declaration scope中的定义。 否则的话，使用scope of the template instantiation

上述例子中，foo的使用，其参数是_val,是个int类型的，与模板实例化的类型参数无关。同时，函数的解析只取决于其签名，也就是其参数类型，跟返回值无关。 因此，解析该函数，只能使用the scope of the template declaration， within this scope is only the one
candidate instance of foo() from which to choose.


Let's look at a type-dependent usage:
```c++
sr0.type_dependent();
```
Which foo() does its call resolve to?
```c++
return foo( _member );
```

This instance clearly is dependent on the template argument that determines the actual type of _member . So this instance of foo() must be resolved from the scope of the template instantiation, which in this case includes both declarations of foo() . Since _member is of type int in this instance, it's the integer instance of foo() that is invoked.

#### **Member Function Instantiation**
一般模板函数的定义被放在头文件中。

有的编译器会在类模板实例化时，其所有的成员函数都会被实例化，有的仅实例化用到的函数，通过simulate linkage of the application to see which instances are actually required and generate only those

如何防止在多个.o文件中实例化相同成员函数？
一种解决方案是生成多个副本，然后提供链接器支持以忽略除一个实例之外的所有实例。
Another solution is the use-directed instantiation strategy of simulating the link phase to determine which instances are required

模板的使用会导致编译时间增加。

程序员可以显示的实例化一个类模板,包括其所有的成员函数
```c++
template class Point3d< float >;
```
显示实例化类模板的某个成员函数
```c++
template float Point3d<float>::X() const;
```
显示实例化某个函数模板
```c++
template Point3d<float> operator+(
const Point3d<float>&, const Point3d<float>& );
```


### **Exception Handling**

#### **Exception Handling Overview**
Exception handling under C++ consists of the following three main syntactic components:
1. A throw clause. A throw clause raises an exception at some point within the program. The exception thrown can be of a built-in or user-defined type.
2. One or more catch clauses. Each catch clause is the exception handler. It indicates a type of exception the clause is prepared to handle and gives the actual handler code enclosed in braces.
3. A try block. A try block surrounds a sequence of statements for which an associated set of catch clauses is active.

When an exception is thrown, control passes up the function call sequence until either an appropriate catch
clause is matched or main() is reached without a handler's being found, at which point the default handler,
terminate(), is invoked. As control passes up the call sequence, each function in turn is popped from the program stack (this process is called unwinding the stack). Prior to the popping of each function, the destructors of the function's local class objects are invoked.

What is slightly nonintuitive about EH is the impact it has on functions that seemingly have nothing to do with exceptions. For example, consider the following:

```c++
1. Point*
2. mumble()
3. {
4.    Point *pt1, *pt2;
5.    pt1 = foo();
6.    if ( !pt1 )
7.      return 0;
8.
9.    Point p;
10.
11.   pt2 = foo();
12.   if ( !pt2 )
13.     return pt1;
14.
15. //...
16 }
```
如果在第一次调用foo()（第5行）时抛出异常，则函数可以从程序堆栈中简单地弹出。该语句不在try块内，因此无需尝试与catch子句匹配；也没有需要销毁的本地类对象。但是，如果在第二次调用foo()（第11行）时抛出异常，则EH机制必须在将函数从程序堆栈展开之前调用p的析构函数。



在EH下，第4-8行和第9-16行被视为函数的语义上不同区域，在抛出异常时具有不同的运行时语义。此外，EH支持需要额外的簿记工作。实现可以将这两个区域分别与要销毁的本地对象列表相关联（这些列表将在编译时设置），或者共享一个在运行时动态添加和缩小的列表。

On the programmer level, EH also alters the semantics of functions that manage resources. The following function includes, for example, both a locking and unlocking of a shared memory region and is no longer guaranteed to run correctly under EH even though it seemingly has nothing to do with exceptions:

```c++
void mumble( void *arena )
{
  Point *p = new Point;
  smLock( arena ); // function call
  // problem if an exception is thrown here
  // ...
  smUnLock( arena ); // function call
  delete p;
}
```

In this case, the EH facility views the entire function as a single region requiring no processing other than unwinding the function from the program stack. Semantically, however, we need to both unlock shared memory and delete p prior to the function being popped. The most straightforward (if not the most effective) method of making the function "exception proof" is to insert a default catch clause, as follows:

```c++
void mumble( void *arena )
{
  Point *p;
  p = new Point;
  try {
    smLock( arena );
    // ...
  }
  catch ( ... ) {
    smUnLock( arena );
    delete p;
    throw;
  }
  smUnLock( arena );
  delete p;
}
```

Notice that the invocation of `operator new` is not within the try block. Is this an error on my part? If either `operator new` or the `Point constructor `invoked after the allocation of memory should throw an exception, neither the unlocking of memory nor the deletion of p following the catch clause is invoked. Is this the correct semantics?

Yes, it is. If operator new throws an exception, memory from the heap would not have been allocated and the Point constructor would not have been invoked. So, there would be no reason to invoke operator delete.

If, however, the exception were thrown within the Point constructor following allocation from the heap, any constructed composite or subobject within Point (that is, a member class or base class object) would automatically be destructed and then the heap memory freed. In either case, there is no need to invoke operator delete. <font color=red> ???</font>

The recommended idiom for handling these sorts of resource management is to encapsulate the resource acquisition within a class object, the destructor of which frees the resource:
```c++
void
mumble( void *arena )
{
auto_ptr <Point> ph ( new Point );
SMLock sm( arena );
// no problem now if an exception is thrown here
// ...
// no need to explicitly unlock & delete
// local destructors invoked here
// sm.SMLock::~SMLock();
// ph.auto_ptr <Point>::~auto_ptr <Point> ()
}
```

#### **Exception Handling Support**
When an exception is thrown, the compilation system must do the following:
1. Examine the function in which the throw occurred.
2. Determine if the throw occurred in a try block.
3. If so, then the compilation system must compare the type of the exception against the type of each catch clause.
4. If the types match, control must pass to the body of the catch clause.
5. If either it is not within a try block or none of the catch clauses match, then the system must (a) destruct any active local objects, (b) unwind the current function from the stack, and (c) go to the next active function on the stack and repeat items 2–5.

##### **Determine if the Throw Occurred within a try Block**
A function, recall, can be thought of as a set of regions:
* A region outside a try block with no active local objects
* A region outside a try block but with one or more active local objects requiring destruction
* A region within an active try block
  
The compiler needs to mark off these regions and make these markings available to the runtime EH system. A predominant strategy for doing this is the construction of program counter range tables.

Recall that the program counter holds the address of the next program instruction to be executed. To mark off a region of the function within an active try block, the beginning and ending program counter value (or the beginning program counter value and its range value) can be stored in a table.

When a throw occurs, the current program counter value is matched against the associated range table to determine whether the region active at the time is within a try block. If it is, the associated catch clauses need to be examined. If the exception is not handled (or if it is rethrown), the current function is popped from the program stack and the value of the program counter is restored to the value of the call site and the cycle begins again.

##### **Compare the Type of the Exception against the Type of Each Catch Clause**
For each throw expression, the compiler must create a type descriptor encoding the type of the exception. If the type is a derived type, the encoding must include information on all of its base class types. 

The type descriptor is necessary because the actual exception is handled at runtime when the object itself otherwise has no type information associated with it. RTTI is a necessary side effect of support for EH.

The compiler must also generate a type descriptor for each catch clause. The runtime exception handler compares the type descriptor of the object thrown with that of each catch clause's type descriptor until either a match is found or the stack has been unwound and terminate() invoked.

An exception table is generated for each function. It describes the regions associated with the function, the location of any necessary cleanup code (invocation of local class object destructors), and the location of
catch clauses if a region is within an active try block.

##### **What Happens When an Actual Object Is Thrown during Program Execution?**
When an exception is thrown, the exception object is created and placed generally on some form of exception data stack. Propagated from the throw site to each catch clause are the address of the exception object, the
type descriptor (or the address of a function that returns the type descriptor object associated with the exception type), and possibly the address of the destructor for the exception object, if one is defined.

Consider a catch clause of the form
```c++
catch( exPoint p )
{
  // do something
  throw;
}
```
and an exception object of type exVertex derived from exPoint. The two types match and the catch clause block becomes active. What happens with p?

* p is initialized by value with the exception object the same as if it were a formal argument of a function. This means a copy constructor and destructor, if defined or synthesized by the compiler, are applied to the local copy.
* Because p is an object and not a reference, the non-exPoint portion of the exception object is sliced off when the values are copied. In addition, if virtual functions are provided for the exception hierarchy, the vptr of p is set to exPoint's virtual table; the exception object's vptr is not copied.


What happens when the exception is rethrown? Is p now the object propagated or the exception object originally generated at the throw site? 

p is a local object destroyed at the close of the catch clause. Throwing p would require the generation of another temporary. It also would mean losing the exVertex portion of the original exception. The original exception object is rethrown; any modifications to p are discarded.

A catch clause of the form
```c++
catch( exPoint &rp )
{
  // do something
  throw;
}
```
refers to the actual exception object. Any virtual invocations resolve to the instances active for exVertex, the actual type of the exception object. Any changes made to the object are propagated to the next catch clause.


Finally, here is an interesting puzzle. If we have the following throw expression:
```c++
exVertex errVer;
// ...
mumble()
{
  // ...
  if ( mumble_cond ) {
    errVer.fileName( "mumble()" );
    throw errVer;
  }
  // ...
}
```
Is the actual exception errVer propagated or is a copy of errVer constructed on the exception stack and propagated? A copy is constructed; the global errVer is not propagated. This means that any changes made to the exception object within a catch clause are local to the copy and are not reflected within errVer. The
actual exception object is destroyed only after the evaluation of a catch clause that does not rethrow the exception.

### **Runtime Type Identification**


#### **Introducing a Type-Safe Downcast**

Downcasts are potentially dangerous because they circumvent the type system and if incorrectly applied may misinterpret (if it's a read operation) or corrupt program memory (if it's a write operation).

One criticism of C++ had been its lack of support for a type-safe downcast mechanism—one that performs the downcast only if the actual type being cast is appropriate . A type-safe downcast requires a runtime query of the pointer as to the actual type of the object it addresses. Thus support for a type-safe downcast mechanism brings with it both space and execution time overhead:

* It requires additional space to store type information, usually a pointer to some type information node.(type_info )

* It requires additional time to determine the runtime type, since, as the name makes explicit, the determination can be done only at runtime.

那么，如果添加这种机制，对于某些不需要多态，不需要cast来cast去的程序而言，以上的overhead不应该存在的。尤其是，以下两种程序
* Programmers who use polymorphism heavily and who therefore have a legitimate need for a type-safe downcast mechanism
* Programmers who use the built-in data types and nonpolymorphic facilities and who therefore have a legitimate need not to be penalized with the overhead of a mechanism that does not come into play in their code

为前者实现RTTI是最好的，但RTTI不应该对后者造成不必要的performance penalty.

The C++ RTTI mechanism provides a type-safe downcast facility but only for those types exhibiting polymorphism . Within C++, a polymorphic class is one that contains either an inherited or declared virtual function

实现上，RTTI就可以利用已经实现了的虚函数机制。每个polymorphism class都会在编译期生成虚函数表，虚函数表的第一个slot存放指向type_info object的指针。这种实现方式的好处是，一是其额外的空间使用很少，每个class多一个指针，和该class对应的type_info oject.此外，在编译期就可以设置好该指针。在运行期，class object被construct时，设置的是class object中的vptr.

#### **A Type-Safe Dynamic Cast**
The dynamic_cast operator determines at runtime the actual type being addressed. If the downcast is safe (that is, if the base type pointer actually addresses an object of the derived class), the operator returns the appropriately cast pointer. If the downcast is not safe, the operator returns 0.
```c++

void Dynamic_Cast( Base * pb )
{
  if ( Derived *pd  = dynamic_cast< Derived * >( pb )) {
      // ... process pd
  }
  else { ... }
}
```
dynamic_cast实际的执行就是首先通过`((type_info*)(pb->vptr[0]))->_type_descriptor`获取pb指向的对象的实际类型的类型描述符，然后获取 Derived type的类型描述符，最后将两者比较，如果相同，则转换成功。

这种做法，相比于直接downcast，是要耗时很多的，但保证了安全性。

#### **References Are Not Pointers**
The dynamic_cast operator can also be applied to a reference. The result of a non–type-safe cast, however,cannot be the same as for a pointer. Why? A reference cannot refer to "no object" the way a pointer does by having its value be set to 0. Initializing a reference with 0 causes a temporary of the referenced type to be generated. This temporary is initialized with 0. The reference is then initialized to alias the temporary. 

Thus the dynamic_cast operator, when applied to a reference, cannot provide an equivalent true/false pair of alternative pathways as it does with a pointer. Rather, the following occurs:
* If the reference is actually referring to the appropriate derived class or an object of a class subsequently derived from that class, the downcast is performed and the program may proceed.
* If the reference is not actually a kind of the derived class, then because returning 0 is not viable, a bad_cast exception is thrown.

```c++
void Dynamic_Cast( Base & pb )
{
  try{
    Derived & pd = dynamic_cast< Derived & >( pb );
    //...
  }
  catch (bad_cast){
    //....
  }
}
```

#### **Typeid Operator**
也可以通过typeid 操作符实现安全的downcast
```c++
void Safe_DownCast( Base & pb )
{
  if ( typeid( Derived ) == typeid( pb ))
  {
      Derived &pd = static_cast< Derived &  >( pb );
      // ...
  }
  else { 
    //... 
  }
}
```
The typeid operator returns a const reference of type type_info. In the previous test, the equality operator is an overloaded instance:
```c++
bool type_info::operator==( const type_info& ) const;
```
and returns true if the two type_info objects are the same.

What does the type_info object consist of? The Standard (Section 18.5.1) defines the type_info class as follows:
```c++
class type_info {
public:
  virtual ~type_info();
  bool operator==( const type_info& ) const;
  bool operator!=( const type_info& ) const;
  bool before( const type_info& ) const;
  const char* name() const;
private:
  // prevent memberwise init and copy
  type_info( const type_info& );
  type_info& operator=( const type_info& );
  // data members
};
```

The minimum information an implementation needs to provide is the actual name of the class, some ordering  algorithm between type_info objects (this is the purpose of the before() member function), and some form of type descriptor representing both the explicit class type and any subtypes of the class. In the original paper describing the EH mechanism (see [KOENIG90b]), a suggested type descriptor implementation is that of an encoded string. (For alternative strategies, see [SUN94a] and [LENKOV92].)

While RTTI as provided by the type_info class is necessary for EH support, in practice it is insufficient to fully support EH. Additional derived type_info classes providing detailed information on pointers, functions, classes, and so on are provided under an EH mechanism. MetaWare, for example, defines the following additional classes:
```c++
class Pointer_type_info: public type_info { ... };
class Member_pointer_info: public type_info { ... };
class Modified_type_info: public type_info { ... };
class Array_type_info: public type_info { ... };
class Func_type_info: public type_info { ... };
class Class_type_info: public type_info { ... };
```
and permits users to access them. Unfortunately, neither the naming conventions nor the extent of these derived classes is standardized, and they vary widely across implementations.

Although I have said that RTTI is available only for polymorphic classes, in practice, type_info objects are also generated for both built-in and nonpolymorphic user-defined types. This is necessary for EH support. For  example, consider
```
int ex_errno;
...
throw ex_errno;
```
where a type_info object supporting the int type is generated. 

对此的支持延伸到用户程序：
```
int *ptr;
...
if ( typeid( ptr ) == typeid( int* ))
   ...
```
Use of `typeid( expression )` within a program, such as
```
int ival;
...
typeid( ival ) ... ;
```
or of `typeid( type )`, such as
```
typeid( double ) ... ;
```
returns a `const type_info&`. The difference between the use of typeid on a `nonpolymorphic expression` or `type` is that the type_info object is retrieved **statically** rather than at runtime. 当然，type_info对象只在需要时才会生成。

### **Efficient, but Inflexible?**
#### **Dynamic Shared Libraries**
通常情况下，在连接新版本的动态共享库时，只需要直接替换库文件就可以了，新的版本的共享库是默认直接连接的。但是，under the C++
Object Model if the data layout of a class object changes in the new library version，则需要重新编译用户代码，
This is because the size of the class and the offset location of each of its direct and inherited members is fixed at compile time (except
for virtually inherited members). This results in efficient but inflexible binaries; a change in the object layout requires recompilation. Both [GOLD94] and [PALAY92] describe interesting efforts in pushing the C++ Object Model to provide increased drop-in support. Of course, the tradeoff is a loss of runtime speed and size efficiency.

#### **Shared Memory**
When a shared library is dynamically loaded, its placement in memory is handled by a runtime linker and generally is of no concern to the executing process. 

This is not true, however, under the C++ Object Model when a class object supported by a dynamically shared library and containing virtual functions is placed in shared memory. 

在虚函数表中，虚函数的地址应该是虚拟地址，因此，如果后续的进程连接该共享库，但是将其映射到不同的虚拟地址处，则会有很大的问题。


The problem is not with the process that is placing the object in shared memory but with a second or any subsequent process wanting to attach to and invoke a virtual function through the shared object. 

Unless the dynamic shared library is loaded at exactly the same virtual memory location as the process that loaded the shared object, the virtual function invocation fails badly. The likely result is either a segment fault or bus error. 

The problem is caused by the hardcoding within the virtual table of each virtual function. 

The current solution is program-based. It is the programmer who must guarantee placement of the shared libraries across processes at the same locations. 

A compilation system-based solution that preserves the efficiency of the virtual table implementation model is required. Whether that will be forthcoming is another issue.

这本书的成书时间是1996年，过去快30年了，以上两个问题应该已经解决了。



