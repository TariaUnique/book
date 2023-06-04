考虑如下的抽象基类
```c++
class Abstract_base {
public:
    virtual ~Abstract_base() = 0;
    virtual void interface() const = 0;
    virtual const char* mumble () const { return _mumble; }
protected:
    char *_mumble;
};
```
虽然这个class被设计为一个抽象的基类（其中有pure virtual function，使得Abstract_base class 不可能拥有object），但它仍然需要一个explicit构造函数以初始化其data member _mmumble.如果没有这个初始化操作，如下的派生类中，_mumble没有被初始化。
```c++
class Concrete_derived : public Abstract_base {
public:
    Concrete_derived();
    // ...
};
void foo()
{
    // Abstract_base::_mumble uninitialized
    Concrete_derived trouble;
    // ...
}
```
如果这个类的设计者的意图就是在派生类的构造函数中初始化基类，同时因为这个类是个虚基类，不应该有public 构造函数，应该在protected中提供显示构造函数，在派生类的构造函数中调用该基类的构造函数。


也有人会说，这个类的设计错误不在于没有提供构造函数初始化其数据，而是在抽象基类中不应该有数据成员，抽象基类只需要提供接口就可以了，将接口与实现分离是一种常见的设计模型。这当然是有道理的，but it does not hold universally. Lifting up data members shared among several derived types can be a legitimate design choice.


### **Presence of a Pure Virtual Desctructor**
C++新手常常很惊讶地发现，一个人竟然可以定义和调用（invoke）一个pure virtual  function：不过它只能被静态地调用（invoked statically），不能经由虚拟机制调用，例如，你可以合法地写下这段码：
```c++
// ok: definition of pure virtual function
// but may only be invoked statically ...
inline void Abstract_base::interface()
{
    // ...
}
inline void Concrete_derived::interface()
{
    // ok: static invocation
    Abstract_base::interface();
    // ...
}
```
要不要这样做，全由class设计者决定．唯一的例外就是pure virtual destructor:class设计者一定得定义它，为什么？因为每一个derived class destructor会被编译器加以扩展，以静态调用的方式调用其每一个base class 的destructor.因此，只要缺乏任何一个base class destructors的定义，就会导致链接失败． 即使基类的虚析构函数被定义为纯虚函数，也需要提供析构函数的定义。


又有人可能会说了，如果一个类的虚析构函数忘记定义了，难道编译器不能合成一个定义么？
因为分离编译的缘故，编译器只有在连接期才可能知道一个类的虚析构函数只有声明没有定义，当然，这时候，也许可以实现一个机制，重新激活编译器，给其一个指令，合成析构函数。但是，应该没有编译器会这么做。 

最好的设计选择是不要将虚析构函数定义为pure virtual.



当然，如果一个类没有声明析构函数，编译器会合成一个析构函数，这个析构函数什么也不做，也不是virtual的。

### **Presence of a Virtual Specification**
将`Abstract_base::mumble()` 声明为virtual 并不是一个好的设计。 
因为显然，这个函数并不是 type dependent, 不太可能在派生类中被重写。并且，这个函数的实现是inline的，而又被声明为virtual,如果一个inline函数被声明为virtual,其inline机制就失效了。 这样，本来因inline而带来的优化就没有了，编译器在遇到mumble函数调用的时候，本来可以原地拓展的，现在却需要走虚函数调用机制。


### **Presence of const within a Virtual Specification**
`Abstract_base::interface()`这个pure virtual函数被声明为const了，
因为虚函数通常是type dependent,意味着派生类很可能要重写该函数。如果将虚函数声明为const，意味着派生类中也需要为const。
如果一个成员函数没有被声明为const, 在const对象调用该成员时需要做 const cast.
而如果一个成员函数被声明为const，实际实现时，比如在派生类中实现时，发现还是需要修改对象成员，这时候只能将const去掉。
所以在基类中声明的虚函数，最好不要声明为const



### **A Reconsidered Class Declaration**
综上所述，以下的Abstract_base的声明应该更合理
```c++
class Abstract_base {
public:
    virtual ~Abstract_base();
    virtual void interface() = 0;
    const char* mumble () const { return _mumble; }
protected:
    Abstract_base( char *pc = 0 );
    char *_mumble;
};
```

### **Object Construction without Inheritance**
考虑如下一段代码
```
(1) Point global;
(2)
(3) Point foobar()
(4) {
(5)     Point local;
(6)     Point *heap = new Point;
(7)     *heap = local;
(8)     // ... stuff ...
(9)     delete heap;
(10)    return local;
(11) }
```
line 1,5,6 代表3个object 变量的创建，global,local,heap memory allocation.

line 7 将一个object赋值给另一个object

line 10 将返回值初始化

line 9 explicitly deletes the heap object

object的生命周期由运行时决定。 The local object's lifetime extends from its definition at line 5 through line 10.
The global object's lifetime extends for the entire program execution. 
The lifetime of an object allocated on the heap extends from the point of its allocation using `operator new` through application of `operator delete`.

假设Point的声明如下
```c++
typedef struct
{
    float x, y, z;
} Point;
```
和C是完全兼容的，这种声明，在C++中被叫做`Plain Ol' Data`

C++在编译这个类时，会发生什么？理论上，编译器会在内部给这个类声明trivial default constructor, trivial destructor, trivial copy constructor, and trivial copy assignment operator 。

而实际上，编译器在分析了这个类之后，将其tag为`Plain Ol' Data`.



当编译器遇到如下的定义时
```c++
(1) Point global;
```
理论上，编译器会定义 Point's trivial constructor and destructor， 并且，在program startup代码中需要调用constructor ，在exit代码中，需要调用destructor。
事实上，这些trivial constructor and destructor 既没有被定义出来，也没有被调用。 the program behaves exactly as it would in C.


只有一点例外，在c语言中，未初始化的全局变量是被放在bss（Block Started by Symbol）段的，bss段在可执行文件中只有符号，其数据是在被加载时零初始化的。
而在c++中，因为构造函数的存在，全局变量都被认为是初始化过了的，不使用bss段，所有的全局变量都被放在已初始化过的全局变量数据段中，当然其中的数据都是零值初始化的。

foobar()函数中的local Point object,没有被初始化，因为构造函数是trivial的，不会做任何事。这样，line 7中对其的直接使用，很可能是个bug

Line 6的
```c++
(6) Point *heap = new Point;
```
is transformed into a call of the library instance of operator new
```c++
Point *heap = __new( sizeof( Point ));
```
Again, there is no default constructor applied to the Point object returned by the call to operator new. The assignment to this object on the next line would solve this problem if local were properly initialized:
```c++
(7) *heap = local;
```
This assignment should generate a compiler warning of the general form
```c++
warning, line 7: local is used before being initialized.
```
理论上， `*heap = local;`编译器在遇到这个语句时，会定义拷贝赋值运算符并调用。而实际上，因为是`Plain Ol' Data`，the assignment remains a bitwise copy exactly as it is in C.

```c++
(9) delete heap;
```
is transformed into a call of the library instance of operator delete
```c++
__delete( heap );
```
Again, conceptually, this triggers the generation of the trivial destructor for Point. But, as we've seen, the destructor, in practice, is neither generated nor invoked. 

Finally, the return of local by value conceptually triggers the definition of the trivial copy constructor, which is then invoked。In practice, the return remains a simple bitwise copy operation of `Plain Ol' Data`.

### **Abstract Data Type**
第二个关于Point的声明如下
```c++
class Point {
public:
    Point( float x = 0.0, float y = 0.0, float z = 0.0 )
    : _x( x ), _y( y ), _z( z ) {}
    // no copy constructor, copy assignment operator
    // or destructor defined ...
    // ...
private:
    float _x, _y, _z;
};
```
提供了对数据的封装，使用ADT设计范式。没有任何virtual函数。

这个类对象的大小同之前一样，只有三个数据成员的大小，其public，private access label不会占用对象任何空间，member function也一样。

我们也没有提供拷贝构造函数，拷贝赋值运算符，default bitwise copy semantics are sufficient. 我们也不会提供析构函数， the default program management of memory is sufficient.

The definition of a global instance
```c++
Point global; // apply Point::Point( 0.0, 0.0, 0.0 );
```
编译器会在程序开始时，给global调用默认构造函数初始化。


如果class的初始化的值都是constant values, 显示初始化列表的使用更高效，如下，local1使用显示初始化列表初始化。而local2使用默认构造函数初始化。
```c++
void mumble()
{
    Point1 local1 = { 1.0, 1.0, 1.0 };
    Point2 local2(1.0,1.0,1.0); 
    // 默认构造函数的 inline expansion 如下
    local2._x = 1.0;
    local2._y = 1.0;
    local2._z = 1.0;

    // the explicit initialization is slightly faster
}
```
local1的初始化比local2更高效， 因为初始化列表中的常量，在mumble函数栈被设置时，就被放置在mumble函数栈中，local1对应的空间中了。
而local2使用默认构造函数初始化，默认构造函数被拓展成一条条需要执行的赋值语句


显示初始化列表初始化也有缺点：
* 只有在所有的class members都是public时，才能使用
* 在初始化列表中只能放置常量表达式(those able to be evaluated at compile time)


The definition of the local Point object
```c++
{
    Point local;
    // ...
}
```
is now followed by the inline expansion of the Point defaul constructor:
```c++
{
  // inline expansion of default constructor
  Point local;
  local._x = 0.0; local._y = 0.0; local._z = 0.0;
  // ...
}
```
The allocation of the Point object on the heap on line 6
```c++
(6) Point *heap = new Point;
```
now includes a conditional invocation of the Point  default  constructor
```c++
// Pseudo C++ Code
Point *heap = __new( sizeof( Point ));
if ( heap != 0 )
    heap->Point::Point();
```
which is then inline expanded. 

The assignment of the local object to the object pointed to by heap
```c++
(7) *heap = local;
```
remains a simple bitwise copy, as does the return of the local object by value :
```c++
(10) return local;
```

The deletion of the object addressed by heap
```c++
(9) delete heap;
```
does not result in a destructor call, since we did not explicitly provide an instance.

Conceptually, our Point class has an associated default copy constructor, copy assignment operator, and destructor. **These, however, are trivial and are not in practice actually generated by the compiler.**

### **Preparing for Inheritance**
下面的第三个Point的声明，提供了面向对象的特性，其中有virtual函数，这个virtual机制主要是针对z data member的存取的
```c++
class Point {
public:
  Point( float x = 0.0, float y = 0.0 ): _x( x ), _y( y ) {}
  // no destructor, copy constructor, or
  // copy  assignment operator defined ...
  virtual float z();
  // ...
protected:
    float _x, _y;
};
```
这里，我们依然没有定义 copy constructor, copy assignment operator, or destructor。 
All our members are stored by value and are therefore well behaved on a program level under the default semantics. (Some people would argue that the introduction of a virtual function should always be accompanied by the declaration of a virtual destructor. But doing that would buy us nothing in this case.) <font clolr=red>这里存疑哈，trivail desctructor是nonvirtual的，如果派生类确实需要在desctrctor中执行一些操作，而 `delete p`将调用 Point的trivial destructor，这是不对的</font>

首先virtual函数的引入，导致每个Point calss object中都会存储一个指向virtual table的指针。其次，编译器会拓展默认构造函数，拷贝构造函数，拷贝赋值运算符
* 对默认构造函数的拓展，需要在其中加上初始化virtual table pointer的代码。 This code has to be added after the invocation of any base class constructors but before execution of any user-supplied code. For example, here is a possible expansion of our Point constructor:
```c++
// Pseudo C++ Code: internal augmentation
Point::Point( Point *this,float x, float y ):_x(x), _y(y)
{
  // set the object's virtual table pointer
  this->__vptr__Point = __vtbl__Point;
  // expand member initialization list
  this->_x = x;
  this->_y = y;
  return;
}
```
* 拷贝构造函数和拷贝赋值运算符都需要被显示合成的，他们现在不是trivial的了。(The implicit destructor remains trivial and so is not synthesized，also it's none virtual)

其合成的函数中，vptr的设置不能简单的拷贝，比如如果是用派生类初始化基类。 其他的成员可能是copy member by member,or chunk by chunk 
```c++
// Pseudo C++ Code:
// internal synthesis of copy constructor
Point::Point( Point *this, const Point &rhs )
{
  // set the object's virtual table pointer
  this->__vptr__Point = __vtbl__Point;
  // 'bitblast' contiguous portion of rhs'coordinates into this object or provide a member by member assignment ...
  return;
}
```

当然，这些函数的合成，都是需要的时候再合成。


line 7的拷贝
```c++
*heap = local;
```
is likely to trigger the actual synthesize of the copy assignment operator and an inline expansion of its invocation, substituting `heap` for the `this` pointer and `local` for the `rhs` argument.

最后line 10， return of local by value，其转换如下
```c++
// Pseudo C++ code: transformation of foobar()
// to support copy construction
void foobar( Point &__result )
{
Point local;
local.Point::Point( 0.0, 0.0 );
// heap remains the same ...
// application of copy constructor
__result.Point::Point( local );
// destruction of local object would go here
// had Point defined a destructor:
// local.Point::~Point();
return;
}
```
如果可以应用NRV优化（前提是合成了拷贝构造函数，但是到目前为止并没有），进一步将拷贝构造函数的调用优化掉
```c++
// Pseudo C++ code: transformation of foobar()
// to support named return value optimization
void foobar( Point &__result )
{
__result.Point::Point( 0.0, 0.0 );
// heap remains the same ...
return;
}
```


In general, if your design includes a number of functions requiring the definition and return of a local class object by value, such as arithmetic operations of the form
```c++
T operator+( const T&, const T& )
{
    T result;
    // ... actual work ...
    return result;
};
```
then it makes good sense to provide a copy constructor even if the default memberwise semantics are sufficient. Its presence triggers the application of the NRV optimization. 

### **Object Construction under Inheritance**
当我们定义一个object如下：
```
T object;
```
实际上会发生什么事情呢？如果T有一个constructor（不论是由user提供或是由编译器合成），它会被调用．但这个constructor内部到底做了哪些事情呢？
1. The data members initialized in the member initialization list have to be entered within the body of the constructor in the order of member declaration.
2. If a member class object is not present in the member initialization list but has an associated default constructor, that default constructor must be invoked.
3. Prior to that, if there is a virtual table pointer (or pointers) contained within the class object, it (they)must be initialized with the address of the appropriate virtual table(s).
4. Prior to that, all immediate base class constructors must be invoked in the order of base class declaration (the order within the member initialization list is not relevant).
   * If the base class is listed within the member initialization list, the explicit arguments, if any, must be passed.
   * If the base class is not listed within the member initialization list, the default constructor (or default memberwise copy constructor) must be invoked, if present.
   * If the base class is a second or subsequent base class, the this pointer must be adjusted.
5. Prior to that, all virtual base class constructors must be invoked in a left-to-right, depth-first search of the inheritance hierarchy defined by the derived class.
   * If the class is listed within the member initialization list, the explicit arguments, if any, must be passed. Otherwise, if there is a default constructor associated with the class, it must be invoked.
   * In addition, the offset of each virtual base class subobject within the class must somehow be made accessible at runtime.
   * These constructors, however, may be invoked if, and only if, the class object represents the "most-derived class." Some mechanism supporting this must be put into place. 


现在Point的声明如下
```c++
class Point {
public:
   Point( float x = 0.0, float y = 0.0 );
   Point( const Point& );
   Point& operator=( const Point& );
   virtual ~Point();
   virtual float z(){ return 0.0; }
   // ...
protected:
   float _x, _y;
};
```
现有Line类声明如下，composed of a begin and end point
```c++
class Line {
Point _begin, _end;
public:
   Line( float=0.0, float=0.0, float=0.0, float=0.0 );
   Line( const Point&, const Point& );
   draw();
   // ...
};
```
Each explicit constructor is augmented to invoke the constructors of its two member class objects. For example, the user constructor
```c++
Line::Line( const Point &begin, const Point &end ): _end( end ), _begin( begin )
{};
```
is internally augmented and transformed into
```c++
// Pseudo C++ Code: Line constructor augmentation
Line::Line( Line *this, const Point &begin, const Point &end ): _end( end ), _begin( begin )
{
this->_begin.Point::Point( begin );
this->_end.Point::Point( end );
return;
};
```

Since the Point class declares a copy constructor, a copy assignment operator, and a destructor (virtual in this case), the implicit copy constructor, copy assignment operator, and destructor for Line are nontrivial.

合成的析构函数不是virtual的(If Line were derived from Point, the synthesized destructor would be virtual. However, because Line contains only Point objects, the synthesized Line destructor is nonvirtual). Within it, the destructors for its two member class objects are invoked in the reverse order of their construction:
```c++
// Pseudo C++ Code: Line destructor synthesis
inline void Line::~Line( Line *this )
{
    this->_end.Point::~Point();
    this->_begin.Point::~Point();
};
```

### **Virtual Inheritance**
```c++
class Point3d : public virtual Point {
public:
   Point3d( float x = 0.0, float y = 0.0, float z = 0.0 )
   : Point( x, y ), _z( z ) {}
   Point3d( const Point3d& rhs )
   : Point( rhs ), _z( rhs._z ) {}
   ~Point3d();
   Point3d& operator=( const Point3d& );
   virtual float z(){ return _z; }
   // ...
protected:
   float _z;
};
```
The conventional constructor augmentation does not work due to the shared nature of the virtual base class:
```c++
// Pseudo C++ Code:
// Invalid Constructor Augmentation
Point3d::Point3d( Point3d *this, float x, float y, float z )
{
   this->Point::Point( x, y );
   this->__vptr__Point3d = __vtbl__Point3d;
   this->__vptr__Point3d__Point =__vtbl__Point3d__Point;
   this->_z = z;
   return this;
}
```

Consider the following three class derivations:
```c++
class Vertex : virtual public Point { ... };
class Vertex3d : public Point3d, public Vertex { ... };
class PVertex : public Vertex3d { ... };
```
The constructor for Vertex must also invoke the Point class constructor. However, when Point3d and Vertex are subobjects of Vertex3d, their invocations of the Point constructor must not occur; rather, Vertex3d, as the most-derived class, becomes responsible for initializing Point. In the subsequent PVertex derivation, it, not Vertex3d, is responsible for the initialization of the shared Point subobject.

这种有时候需要调用virtual base class的构造函数，有时候又不需要，通常是通过对构造函数引入一个参数来实现的。 如下
```c++
// Psuedo C++ Code:
// Constructor Augmentation with Virtual Base class
Point3d::Point3d( Point3d *this, bool __most_derived,
float x, float y, float z )
{
if ( __most_derived != false )
    this->Point::Point( x, y);
this->__vptr__Point3d = __vtbl__Point3d;
this->__vptr__Point3d__Point = __vtbl__Point3d__Point;
this->_z =z;
return;
}
```
后续派生的类，比如Vertex3d,其构造函数中，在调用Point3d 和 Vertex的构造函数时，都将__most_derived参数设置为false,suppressing the Point constructor invocation within both constructors
```c++
// Psuedo C++ Code:
// Constructor Augmentation with Virtual Base class
Vertex3d::Vertex3d( Vertex3d *this, bool __most_derived,
float x, float y, float z )
{
if ( __most_derived != false )
    this->Point::Point( x, y);
// invoke immediate base classes,
// setting __most_derived to false
this->Point3d::Point3d( false, x, y, z );
this->Vertex::Vertex( false, x, y );
// set vptrs...
// insert user code....
return;
}
```

还有一种更为有效的实现，就是将constructor生成两份，一份用于完整对象的调用，其中会调用其virtual base class 的 constructor,sets all vptrs, and so on.另一份针对subobject， does not invoke the virtual base class constructors, may possibly not set the vptrs, and so on 。不过作者没有提供哪种编译器使用这种方式实现的。


### **The Semantics of the vptr Initialization**
When we define a PVertex object, the order of constructor calls is
```c++
Point( x, y );
Point3d( x, y, z );
Vertex( x, y, z );
Vertex3d( x, y, z );
PVertex( x, y, z );
```
假设这个继承体系中的每一个类中都声明了virtual function size() that returns the size in bytes of the class。下面代码
```c++
PVertex pv;
Point3d p3d;
Point *pt = &pv;
```
the call
```
pt->size();
```
would return the size of the PVertex class and
```
pt = &p3d
pt->size();
```
would return the size of the Point3d class.

在上述继承体系的类中，每个类的构造函数中都有对size的调用，如下
```c++
Point3d::Point3d( float x, float y, float z ) : _x( x ), _y( y ), _z( z )
{
   if ( spyOn )
    cerr << "within Point3d::Point3d()"<< " size: " << size() << endl;
}
```
这样，PVertex的构造函数中，其各基类的构造函数的调用，其中对size的解析，是解析到PVertex::size()，还是解析到正在被构造的subobject的size呢？答案是解析到正在被构造的subobject的size。这是必须保证的，因为类的构造是从基类到派生类，在构造基类时，派生类还不完整，不能调用派生类的虚函数。但这是如何实现的？

如果需要在构造函数或析构函数中调用虚函数，应该直接静态调用，比如上面的Point3d构造函数中对size的调用应该如下
```c++
Point3d::size() 
```
然而如果size函数中又有对虚函数的调用，这时候只能走虚函数机制调用了。虚函数的调用主要通过vptr,因此vptr的初始化时机就很重要了。vptr的初始化时机，**After invocation of the base class constructors but before execution of user-provided code or the expansion of members initialized within the member initialization list**

一个构造函数的执行流程如下
* Within the derived class constructor, all virtual base class and then immediate base class constructors are invoked.
* That done, the object's vptr(s) are initialized to address the associated virtual table(s).
* The member initialization list, if present, is expanded within the body of the constructor. This must be done after the vptr is set in case a virtual member function is called.
* The explicit user-supplied code is executed.

假设PVertex的构造函数如下
```c++
PVertex::PVertex( float x, float y, float z ): _next( 0 ), Vertex3d( x, y, z ), Point( x, y )
{
if ( spyOn )
  cerr << "within Point3d::Point3d()"
  << " size: " << size() << endl;
}
```
按照如上的流程，会被拓展成
```c++
// Pseudo C++ Code
// expansion of PVertex constructor
PVertex::PVertex( Pvertex* this, bool __most_derived, float x, float y, float z )
{
// conditionally invoke the virtual base constructor
if ( __most_derived != false )
    this->Point::Point( x, y );
// unconditional invocation of immediate base
this->Vertex3d::Vertex3d( false,x, y, z );
// initialize associated vptrs
this->__vptr__PVertex = __vtbl__PVertex;
this->__vptr__Point__PVertex = __vtbl__Point__PVertex;

this->_next=0;
// explicit user code
if ( spyOn )
  cerr << "within PVertex::PVertex()"
  << " size: "
  // invocation through virtual mechanism
  << (*this->__vptr__PVertex[ 3 ].faddr)(this)
  << endl;
return ;
}
```
然而这个解决方案中，所有的base class 构造函数中都设置其vptr,而这个设置的值，很可能会在其下一层的派生类的构造函数中再次被设置。

另一种构造函数vptr的设置，就是将构造函数实现成2分，一份针对完整的对象的构造函数实现，其中需要设置所有的vptr,另一个针对subobject的构造实现，其中vptr的设置，能省略就被省略。除非该构造函数中调用了虚函数，则vpr需要被设置。


下面回答两个问题：

* 在类的构造函数中，在其初始化列表中调用virtual function作为其数据成员的初始值，是否可以？

如果仅从virtual function调用到正确的虚函数这一调度考虑，是可以的。因为 vptr is guaranteed to have been set by the compiler prior to the expansion of the member initialization list.
而如果该函数中，需要访问未被初始化的数据成员，就是错误的了。总的来说，这种idiom是不推荐的。

* 如果是调用virtual function作为基类的构造函数参数呢？
  
这样绝对是错误的， The vptr is either not set or set to the wrong class.Further, any of the data members of the class that are accessed within the function are guaranteed to not yet be initialized.


### **Object Copy Semantics**
当我们设计一个class，并以一个class object赋值给另一个class object时，
我们有三种选择：
1. 什么都不做，因此得以实施默认行为。
2. 提供一个explicit copy assignment operator
3. 明确地拒绝把一个class object 赋值给另一个class object

如果我们要实现第三点，那么可以将copy assignment operator声明为private,并且不提供其定义。
把它设为private，我们就不再允许于任何地点（除了在member functions以及此class
的friends之中）进行賦值（assign）操作．不提供其函数定义，则一旦某个member
function或friends企图调用赋值操作，程序在链接时就会失败

假设Point类声明如下
```c++
class Point {
public:
   Point( float x = 0.0, y = 0.0 );
   //...( no virtual functions
protected:
   float _x, _y;
};
```
没有什么理由需要禁止拷贝一个Point object.因此问题就变成了：默认行为是否足够？如果我们要支持的只是一个简单的拷贝操作，那么默认行为不但足够而且有效率，我们没有理由再自己提供一copy assignment operator

如果我们不对供应一个copy assignment operator，而光是依赖默认的memberwise copy，编译器会产生出一个copy assignment operator实体吗？这个答案和copy constructor的情况一样：实际上不会！由于此class已经有了bitwise copy语意，所以implicit copy assignment operator被视为毫无用处，也根本不会被合成出来．

一个class在以下几种情况下不具有bitwise copy semantics:
* When the class contains a member object of a class for which a copy assignment operator exists
* When the class is derived from a base class for which a copy assignment operator exists
* When the class declares one or more virtual functions (we must not copy the vptr address of the righthand class object, since it might be a derived class object)
* When the class inherits from a virtual base class

如果copy assignment operator不具有bitwise copy semantics时，copy assignment operator是nontrivial的，编译器会合成出具体的实例。


For our Point class, then, the assignment
```c++
Point a, b;
...
a = b;
```
is accomplished as a bitwise copy of Point b into Point a ; no copy assignment operator is invoked.



假设我们提供了如下的copy assignment operator:
```c++
inline Point& Point::operator=( const Point &p )
{
  _x = p._x;
  _y = p._y;
}
```

Now let's derive our Point3d class (note the virtual inheritance):
```c++
class Point3d : virtual public Point {
public:
  Point3d( float x = 0.0, y = 0.0, float z = 0.0 );
  ...
protected:
  float _z;
}
```
If we do not define a copy assignment operator for Point3d, the compiler needs to synthesize one . The synthesized instance might look as follows: 
```c++
// Pseudo C++ Code: synthesized copy assignment operator
inline Point3d& Point3d::operator=( Point3d *const this, const Point3d &p )
{
  // invoke the base class instance
  this->Point::operator=( p );
  // memberwise copy the derived class members
  _z = p._z;
  return *this;
}
```
那么假设Vertex也虚拟继承自Point,其合成的拷贝构造函数如下
```c++
// class Vertex : virtual public Point
inline Vertex& Vertex::operator=( const Vertex &v )
{
  this->Point::operator=( v );
  _next = v._next;
  return *this;
}
```

这时候，Vertex3d继承自Point3d和Vertex，那么合成的拷贝构造函数是不是就是如下呢？
```c++
inline Vertex3d& Vertex3d::operator=( const Vertex3d &v )
{
  this->Point::operator=( v );
  this->Point3d::operator=( v );
  this->Vertex::operator=( v );
  ...
}
```
这看上去是不对的，因为`this->Point3d::operator=( v );`和`this->Vertex::operator=( v );`中都调用了`this->Point::operator=( v );`,我们似乎应该像处理构造函数那样，给拷贝构造函数加上参数，只有most derived class中才会调用`this->Point::operator=( v );`,而中间的`Point3d::operator=`和`Vertex::operator=`不调用`Point::operator=()`; 但这种方法却不能被使用。因为不能给拷贝构造函数加额外参数。

Actually, the copy assignment operator is ill behaved under virtual inheritance and needs to be carefully
designed and documented. In practice, many compilers don't even try to get the semantics right. They invoke each virtual base instance within each intermediate copy assignment operator, thus causing multiple instances of the virtual base class copy assignment operator to be invoked. Cfront does this as well as the Edison Design Group's front-end, Borland's 4.5 C++ compiler, and Symantec's latest C++ Compiler under Windows. My guess is your compiler does it as well. What does the Standard have to say about this?
**It is unspecified whether subojects representing virtual base classes are assigned more than once by the implicitly defined copy assignment operator (Section 12.8)**

所以在使用virtual base class时要慎重了


One way to ensure the most-derived class effects the virtual base class subobject copy is to place an explicit call of that operator last in the derived class instance of the copy assignment operator:
```c++
inline Vertex3d&
Vertex3d::operator=( const Vertex3d &v )
{
this->Point3d::operator=( v );
this->Vertex::operator=( v );
// must place this last if your compiler does
// not suppress intermediate class invocations
this->Point::operator=( v );
...
}
```
This doesn't elide the multiple copies of the subobject, but it does guarantee the correct final semantics.

我建议尽可能不要允许一个virtual base class的拷贝操作．更进一步：不要在任何virtual base class中声明数据。

### **Semantics of Destruction**
如果class没有声明destructor,如果该类其member or base class 有destructor, 编译器才会給其合成析构函数。否则，the destructor is considered to be trivial and is therefore neither synthesized nor invoked in practice.

如下的Point class，编译器时不会给其合成析构函数的，尽管其有virtual function
```c++
class Point {
public:
  Point( float x = 0.0, float y = 0.0 );
  Point( const Point& );
  virtual float z();
  // ...
private:
  float _x, _y;
};
```

如下的Line类，编译器也不会给其合成析构函数，因为Point类没有析构函数
```c++
class Line {
public:
  Line( const Point&, const Point& );
  // ...
  virtual draw();
  // ...
protected:
  Point _begin, _end;
};
```
当我们从Point派生出Point3d,哪怕是虚拟派生，如果我们没有给Point3d声明一个destructor，编译器也不会为其合成一个destructor．

考虑如下代码
```c++
{
  Point pt;
  Point *p = new Point3d;
  foo( &pt, p );
  ...
  delete p;
}
```
在调用`foo( &pt, p );`之前，pt 和 p指向的Point3d都必须初始化好，因此，有必要为其提供构造函数。
否则的话，其值不仅没有初始化，而且是随机的，并不是零值，也就无法根据其是否是零值来判断是否初始化过了。

而最后的`delete p`,需不需要提供析构函数呢？object占用的内存直接释放掉就可以了，并不需要对这块内存做任何特殊处理，所以不需要析构函数。 

假设我们有个Vertex类，数据成员是vertices对象的list，析构时，需要按照顺序遍历链表，并释放每一个vertices.因此，其析构函数是显示提供的。
如果Vertex3d派生自Point3d和Vertex,如果我们没有给Vertex3d显示提供析构函数，编译器会合成析构函数，在其中调用Vertex的析构函数。

If we provide a Vertex3d destructor, the compiler augments it to invoke the Vertex destructor after the user supplied code is executed. A user-defined destructor is augmented in much the same way as are the constructors, except in reverse order:

1. If the object contains a vptr, it is reset to the virtual table associated with the class.
2. The body of the destructor is then executed; that is, the vptr is reset prior to evaluating the user-supplied code.
3. If the class has member class objects with destructors, these are invoked in the reverse order of their declaration.
4. If there are any immediate nonvirtual base classes with destructors, these are invoked in the reverse order of their declaration.
5. If there are any virtual base classes with destructors and this class represents the most-derived class, these are invoked in the reverse order of their original construction.


一个派生对象在析构过程中，随着从派生类到基类的析构函数被调用，其对象类型事实上也变成正在被释放的基类的类型。 A PVertex object, for example, becomes in turn a Vertex3d object, a Vertex object, a Point3d object， and then a Point object before its actual storage is reclaimed.

每一层的析构函数中，首先都是reset vptr pointer，如果析构函数中有virtual function被调用，都是调用的当前正在被析构的类型的virtual function.




