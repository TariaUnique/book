假设我们有如下简单的表达式
```c++
if ( yy == xx.getValue() ) ...
```
where xx and yy are defined as
```c++
X xx;
Y yy;
```
class Y is defined as
```c++
class Y {
public:
    Y();
    ~Y();
    operator==( const Y& ) const;
    // ...
};
```
and class X is defined as
```c++
class X {
public:
    X();
    ~X();
    operator Y() const;
    X getValue();
    // ...
};
```
这个表达式如何被处理？
第一步，==运算符调用的是`Y::operator==()`,转换如下
```c++
// resolution of intended operator
if ( yy.operator==( xx.getValue() ))
```
getValue()返回的是type X,而Y::operator==()需要的参数类型是type Y,因此，需要调用X::operator Y()转换操作符，第二步的转换如下
```c++
// conversion of getValue()'s return value
if ( yy.operator==( xx.getValue().operator Y() ))
```

到目前位置，都是编译器直接对我们代码的拓展。我们也可以直接写成上述的格式，但这个是不推荐的。

Although the program text is semantically correct, it is not yet instructionally correct. Next we must generate temporaries to hold the return values of our function calls:
1. Generate a temporary of class X to hold the return value of getValue() :
```c++
X temp1 = xx.getValue();
```
2. Generate a temporary of class Y to hold the return value of operator Y() :
```c++
Y temp2 = temp1.operator Y();
```
3. Generate a temporary of type int to hold the return value of the equality operator:
```c++
int temp3 = yy.operator==( temp2 );
```
4. Finally, the language requires that we apply the appropriate destructor to each class object temporary. 

综上，最终转换成的代码如下
```c++
// Pseudo C++ code
// transformation of conditional expression:
// if ( yy == xx.getValue() ) ...
{
X temp1 = xx.getValue();
Y temp2 = temp1.operator Y();
int temp3 = yy.operator==( temp2 );
if ( temp3 ) ...
temp2.Y::~Y();
temp1.X::~X();
}
```

### **Object Construction and Destruction**
通常，构造函数和析构函数的插入如下
```c++
// Pseudo C++ Code
{
Point point;
// point.Point::Point() generally inserted here
...
// point.Point::~Point() generally inserted here
}
```
如果一个代码块{}或函数中有一个以上的离开点，情况会稍微混乱一些，Destructor必须被放在每一个离开点之前，例如：
```c++
{
   Point point;
   // constructor goes here ...
   switch( int( point.x() )) {
      case -1:
         // mumble;
         // destructor goes here
         return;
      case 0:
         // mumble;
         // destructor goes here
         return;
      case 1:
         // mumble;
         // destructor goes here
         return;
      default:
         // mumble;
         // destructor goes here
         return;
   }
   // destructor goes here
}
```
同样的道理，goto指令也可能导致需要多个destructor调用操作，例如下面的
程序片段：
```c++
{
   if ( cache )
      // check cache; if match, return 1
   Point xx;
   // constructor goes here
   while ( cvs.iter( xx ))
      if ( xx == value )
        goto found;
   // destructor goes here
   return 0;
   found:
      // cache item
      // destructor goes here
      return 1;
}
```

### **Global Objects**
如下代码
```c++
Matrix identity;
main()
{
   // identity must be initialized by this point!
   Matrix m1 = identity;
   ...
   return 0;
}
```
如果identity有构造函数和析构函数的话，the language guarantees that identity is constructed prior to the first user statement of main() and destructed following the last statement of main() .

A global object such as identity with an associated constructor and destructor is said to require both static initialization and deallocation.

C++所有的全局变量都被放在数据段，要么有初值，要么是零值。

Thus in this code fragment:
```c++
int v1 = 1024;
int v2;
```
数据段中，v1的值是1024，v2的值是0.（注意v2并不是放在bss段，c++所有全局变量都在数据段，不管有没有初始值）

当然，如果全局变量是对象，构造函数并不是const expresssion,不能在编译期求值。Although the class object can be placed within the data segment during compilation with its memory zeroed out, its constructor cannot be applied until program startup.

在进程startup时，给data segment中的object调用构造函数初始化全局对象，就是所谓的static initialization

cfront 编译器对static initialization and deallocation的实现如下
* 在需要静态初始化的每个文件中，生成一个包含必要构造函数调用的__sti()函数。例如，identity会导致在文件matrix.c中生成以下__sti()函数：
```c++
__sti__matrix_c__identity() {
// Pseudo C++ Code
identity.Matrix::Matrix();
}
```
其中，__matrix_c 是文件名的编码，而 __identity 则代表文件中定义的第一个非静态对象。将这两个名称附加到 __sti 上可在可执行文件中提供唯一标识符。
* 同样地，在每个需要静态释放的文件中，生成一个包含必要析构函数调用的__std()函数。在我们的例子中，会生成一个__std()函数来调用Matrix析构函数并释放identity。
* _main()函数来调用可执行文件中的所有__sti()函数，exit()函数类似地调用所有__std()函数。

cfront inserted a `_main()` call as the new first statement within main() . The `exit()` function rather than the C library `exit()` function was linked in by cfront's `CC` command、

需要解决的最后一个问题是如何收集所有.o文件中的__sti()和__std()函数，看不懂，略

以上的cfront的最初实现版本叫做munch，后来新的实现叫做patch，具体略。这两个实现版本都是通用的。

一旦特定于平台的C++编译器开始出现，通过扩展link editor和目标文件格式以直接支持静态初始化和释放，就可以实现更高效的方法。例如，the System V Executable and Linking Format (ELF) 被扩展为提供.init和.fini sections ，其中包含有关需要进行静态初始化和释放的对象的信息。

Implementation-specific startup routines (usually named something like crt0.o ) complete the platform-specific support for static initialization and deallocation.

### **Local Static Objects**
如下代码中有static local object mat_identity
```c++
const Matrix& identity() {
  static Matrix mat_identity;
  // ...
  return mat_identity;
}
```
static local object有如下guaranteed semantics 

* mat_identity must have its constructor applied only once, although the function may be invoked multiple times.

* mat_identity must have its destructor applied only once, although again the function may be invoked multiple times.

一种实现策略是在程序启动时无条件构造对象。然而，这会导致所有static local objects在程序启动时初始化，无论它们的相关函数是否被使用。相反，更好的方法是仅在第一次调用identity()时构造mat_identity（这现在是标准C++所要求的）。我们该如何做到这一点？

在cfront中，引入了一个临时变量来保护mat_identity的初始化。第一次调用identity()时，临时变量计算为false。然后调用构造函数，并将临时变量设置为true。这解决了构造问题。

析构函数需要有条件地应用于mat_identity，但仅当mat_identity已被构建时才需要应用。确定它是否已经被构建很简单：如果临时变量为真，则它已经被构建了。

static local object的析构应该在程序退出时，问题在于mat_identity was  local to the function，如何在函数外访问它？cfront 采用的方法是获取其地址。 static local object的地址在数据段中。


cfront输出（稍微美化一下）：
```c++
// generated temporary static object guard
static struct Matrix *__0__F3 = 0 ;
// the C analog to a reference is a pointer
// identity()'s name is mangled based on signature
struct Matrix* identity__Fv ()
{
// the __1 reflects the lexical level
// this permitted support for code such as
// int val;
// int f() { int val;
// return val + ::val; }
// where the last line becomes
// ....return __1val + val;
static struct Matrix __1mat_identity ;
// if the guard is set, do nothing, else
// (a) invoke the constructor: __ct__6MatrixFv
// (b) set the guard to address the object
__0__F3
? 0
:(__ct__1MatrixFv ( & __1mat_identity ),
(__0__F3 = (&__1mat_identity)));
...
}
```
Finally, the destructor needed to be conditionally invoked within the static deallocation function associated with the text program file, in this case stat_0.c:
```c++
char __std__stat_0_c_j ()
{
  __0__F3
  ? __dt__6MatrixFv( __0__F3 , 2)
  : 0 ;
  ...
}
```

### **Arrays of Objects**
假设我们有如下的数组定义
```c++
Point knots[ 10 ];
```
如果Point既没有构造函数也没有析构函数，那么我们所需要做的和为内置类型数组一样分配足够存储10个连续的Point元素的内存空间。

如果Point定义了默认构造函数，必须逐个应用于每个元素。通常情况下，这是通过一个或多个运行时库函数来完成的。在cfront中，我们使用了一个名为vec_new()的函数实例来支持创建和初始化类对象数组。更近期的一些实现中，在处理不含虚基类的类和含有虚基类的类时提供两个实例——一个通常被命名为vec_vnew() 。

其签名通常如下，尽管在各种实现中存在变体：
```c++
void* vec_new(
void *array, // address of start of array
size_t elem_size, // size of each class object
int elem_count, // number of elements in array
void (*constructor)( void* ),
void (*destructor)( void*, char )
)
```
其中，constructor和destructor分别是指向类的默认构造函数和析构函数的指针。array存储数组的地址或者为0。如果为0，则表示该数组通过应用 operator new 在堆上进行动态分配。elem_size参数表示数组中元素的数量。（在第6.2节中，我将讨论new和delete运算符。）在vec_new()函数内部，构造函数依次应用于elem_count个元素。析构函数对于异常处理的支持是必要的(it must be applied in the event the constructor being applied throws an exception)。Here is a likely compiler invocation of vec_new() for our array of ten Point elements:
```c++
Point knots[ 10 ];
vec_new( &knots, sizeof( Point ), 10, &Point::Point, 0 );
```
如果Point也定义了析构函数，那么在knots的生命周期结束时需要将其应用于每个元素。毫不奇怪，这是通过类似的vec_delete()（或对于具有虚基类的类而言是vec_vdelete()）运行时库函数来实现的。它通常具有以下签名：
```c++
void*
vec_delete(
void *array, // address of start of array
size_t elem_size, // size of each class object
int elem_count, // number of elements in array
void (*destructor)( void*, char )
)
```

What if the programmer provides one or more explicit initial values for an array of class objects, such as the following:
```c++
Point knots[ 10 ] = {
Point(),
Point( 1.0, 1.0, 0.5 ),
-1.0
};
```
For those elements explicitly provided with an initial value, the use of vec_new() is unnecessary. For the remaining uninitialized elements, vec_new() is applied the same as for an array of class elements without an explicit initialization list. The previous definition is likely to be translated as follows:
```c++
Point knots[ 10 ];
// Pseudo C++ Code
// initialize the first 3 with explicit invocations
Point::Point( &knots[0]);
Point::Point( &knots[1], 1.0, 1.0, 0.5 );
Point::Point( &knots[2], -1.0, 0.0, 0.0 );
// initialize last 7 with vec_new ...
vec_new( &knots+3, sizeof( Point ), 7, &Point::Point, 0 );
```

### **Default Constructors and Arrays**
程序员不允许获取构造函数的地址。当然，这正是编译器在支持vec_new()时所做的。但是，通过指针调用构造函数时，如果构造函数有默认参数，这个默认参数如何获知？

在cfront 2.0版本之前，声明一个类对象的数组意味着该类必须没有构造函数或者只有一个不带参数的默认构造函数。不允许使用带有一个或多个默认参数的构造函数。

在cfront2.0之后，这个不合理的限制被修改了。

cfront会生成一个不带参数的an internal stub constructor 。在函数体内，调用用户提供的构造函数，并将默认参数显式地指定。 Of course, the stub instance is generated and used only if an array of class objects is actually created.

### **Operators new and delete**
```c++
int *pi = new int( 5 );
```
it is actually accomplished as two discrete steps:
* The allocation of the requested memory through invocation of the appropriate operator new instance, passing it the size of the object:
```c++
// invoke library instance of operator new
int *pi = __new ( sizeof( int ));
```
* The initialization of the allocated object:
```c++
*pi = 5;
```

Further, the initialization is performed only if the allocation of the object by operator new succeeds:
```c++
// discrete steps of operator new
// given: int *pi = new int( 5 );
// rewrite declaration
int *pi;
if ( pi = __new( sizeof( int )))
    *pi = 5;
```
Operator delete is handled similarly. When the programmer writes
```c++
delete pi;
```
the language requires that operator delete not be applied if pi should be set to 0. Thus the compiler must construct a guard around the call:
```c++
if ( pi != 0 )
    __delete( pi );
```
Note that pi is not automatically reset to 0 and a subsequent dereference, such as
```c++
if ( pi && *pi == 5 ) ...
```
may or may not evaluate as true. This is because an actual alteration or reuse of the storage that pi addresses may or may not have occurred.

The allocation of a class object with an associated constructor is handled similarly. For example,
```c++
Point3d *origin = new Point3d;
```
is transformed into
```c++
Point3d *origin;
//Pseudo C++ code
if ( origin = __new( sizeof( Point3d )))
    origin = Point3d::Point3d( origin );
```
If exception handling is implemented, the transformation becomes somewhat more complicated:
```c++
// Pseudo 3.C++ code
if ( origin = __new( sizeof( Point3d ))) {
  try {
      origin = Point3d::Point3d( origin );
  }
  catch( ... ) {
      // invoke delete lib function to
      // free memory allocated by new ...
      __delete( origin );
      // propagate original exception upward
      throw;
  }
}
```

The application of the destructor is similar. The expression
```c++
delete origin;
```
becomes
```c++
if ( origin != 0 ) {
  // Pseudo C++ code
  Point3d::~Point3d( origin );
  __delete( origin );
}
```
Under exception handling, the destructor would be placed around a try block. The exception handler would invoke the delete operator and then rethrow the exception.

The general library implementation of operator new is relatively straightforward(Note: The following version does not account for exception handling.)
```c++
extern void* operator new( size_t size )
{
  if ( size == 0 )
    size = 1;
  void *last_alloc;
  while ( !( last_alloc = malloc( size )))
  {
    if ( _new_handler )
        ( *_new_handler )();
    else 
      return 0;
  }
  return last_alloc;
}
```

Although it is legal to write
```c++
new T[ 0 ];
```
但通过上述的operator new的实现可见，即使数组大小是0， operator new都会至少返回一个指向默认 1 字节内存块的指针

上述的operator new的实现中，_new_handler()是需要用户提供的，如果用户没有提供，则为null.

实际上，operator new 一直是通过标准的 C malloc() 实现的，同样地，实际上 operator delete 也一直是通过标准的 C free() 实现的。
```c++
extern void operator delete( void *ptr )
{
if ( ptr )
    free( (char*) ptr );
}
```

### **The Semantics of new Arrays**
当我们写如下代码
```c++
int *p_array = new int[ 5 ];
```
vec_new() 实际上并没有被调用，因为它的主要功能是将默认构造函数应用于类对象数组中的每个元素。Rather, the operator new instance is invoked:
```c++
int *p_array = (int*) __new( 5 * sizeof( int ));
```
Similarly, if we write
```c++
// struct simple_aggr { float f1, f2; };
simple_aggr *p_aggr = new simple_aggr[ 5 ];
```
同上一样，vec_new() 实际上并没有被调用。因为 simple_aggr 没有定义构造函数或析构函数，数组的分配和删除只涉及实际存储空间的获取和释放。

If a class defines a default constructor, however, some version of vec_new() is invoked to allocate and construct the array of class objects. For example, the expression
```c++
Point3d *p_array = new Point3d[ 10 ];
```
is generally transformed into
```c++
Point3d *p_array;
p_array = vec_new( 0, sizeof( Point3d ), 10, &Point3d::Point3d,&Point3d::~Point3d );
```
这里传递析构函数，是为了在构造过程中如果出现异常，则已经构造完成的对象需要被释放。

在2.0版本之前，程序员需要负责提供应用operator delete删除的数组的实际大小。因此，之前的分配，其释放应该写成
```c++
delete [10] p_array;
```
在2.1版本中，语言已经修改。用户不再需要指定要删除的数组元素数目。因此我们现在可以写成：
```c++
delete [] p_array;
```
cfront实现了 the storage and then the retrieval of the element count associated with the pointer.

在delete时，编译器仅在方括号存在时才搜索维度大小。否则，它会假定正在删除单个对象。如果程序员未提供必要的方括号，则会出现错误，只有数组的第一个元素被析构，其余的元素未被析构，尽管它们关联的内存已经被回收。

The less error-prone strategy of checking all delete operations for a possible array entry was rejected as being too costly.

the storage and then the retrieval of the element count associated with the pointer 是怎么实现的呢？

一个显而易见的方法是，为vec_new()操作符返回的每个内存块分配一个额外的字，并将element count存储在那个字中（通常，藏起来的值被称为cookie）。然而，Jonathan和Sun实现选择keep an associative array of pointer values and the array size.

cookie strategy的问题在于，if a bad pointer value should get passed to delete_vec()，the cookie fetched would of course be invalid. An invalid element count and bad beginning address would result in the destructor's being applied on an arbitrary memory arena arbitrary number of times.Under the associative array strategy, however, the likely result of a bad address's being passed is simply the failure to fetch an element count

我们大概看下关联数组的实现：

Within the original implementation, two primary functions were added to support the storage and retrieval of the element count 

```c++
// array_key is the address of the new array
// mustn't either be 0 or already entered
// elem_count is the count; it may be 0
typedef void *PV;
extern int __insert_new_array(PV array_key, int elem_count);
// fetches (and removes) the array_key from table
// either returns the elem_count, or -1
extern int __remove_old_array(PV array_key);

```
original cfront implementation of vec_new() 
```c++
PV __vec_new(PV ptr_array, int elem_count, int size, PV construct )
{
  // if ptr_array is 0, allocate array from heap
  // if set, programmer wrote either
  // T array[ count ];
  // or
  // new ( ptr_array ) T[ 10 ]
  int alloc = 0; // did we allocate here within vec_new?
  int array_sz = elem_count * size;
  if ( alloc = ptr_array == 0)
    // global operator new ...
    ptr_array = PV( new char[ array_sz ] );
  // under Exception Handling,
  // would throw exception bad_alloc
  if ( ptr_array == 0)
      return 0;
  // place (array, count) into the cache
  int status = __insert_new_array( ptr_array, elem_count );
  if (status == -1) {
    // under Exception Handling, would throw exception
    // would throw exception bad_alloc
    if ( alloc )
      delete ptr_array;
    return 0;
  }
  if (construct) {
      char* elem = (char*) ptr_array;
      char* lim = elem + array_sz;
      // PF is a typedef for a pointer to function
      PF fp = PF(construct);
      while (elem < lim) {
        // invoke constructor through fp
        // on `this' element addressed by elem
        (*fp)( (void*)elem );
        // then advance to the next element
        elem += size;
      }
  }
  return PV(ptr_array);
}
```
vec_delete() works similarly, but its behavior is not always what the C++ programmer either expects or requires. For example, given the following two class declarations:
```c++
class Point { 
public:
  Point();
  virtual ~Point();
  // ...
};
class Point3d : public Point { 
public:
  Point3d();
  ~Point3d();
  // ...
}
```

the allocation of an array of ten Point3d objects results in the expected invocation of both the Point and Point3d constructor ten times, once for each element of the array:
```c++
// Not at all a good idea
Point *ptr = new Point3d[ 10 ];
```
而当我们如下调用delete时
```c++
// oops: not what we need!
// only Point::~Point invoked ...
delete [] ptr;
```
传递给vec_delete()的析构函数，是根据指针类型来的，也就是传递的是Point::~Point。这样，不仅调用的析构函数不对，而且，传递给vec_delete()的元素大小，也是根据指针指向的对象大小来的，也不对。 Oops. The whole operation fails miserably. Not only is the wrong constructor applied, but after the first element, it is applied to incorrect chunks of memory.

所以，直接使用派生类的指针来访问派生数组。
```c++
Point3d *ptr = new Point3d[ 10 ];
```

书中所写的另一种方法，针对确实使用`Point *ptr = new Point3d[ 10 ];`Point指针存储返回的数组地址的情况
```c++
for ( int ix = 0; ix < elem_count; ++ix )
{
Point *p = &((Point3d*)ptr)[ ix ];
delete p;
}
```
虽然对Point3d::~Point3d()的调用没问题了，但是在第一次delete时，整块数组的内存就被释放回malloc库了，后续的delete, 又在调用free释放回malloc库，这样会导致错误。


### **Temporary Objects**
假设有关于class T的加法操作
```c++
T operator+( const T&, const T& );
```
假设有如下表达式
```c++
a+b;
```
a,b是class T, 该表达式的计算是否会产生栈外临时对象保存结果，一般来说视表达式的上下文和编译器的优化等级而定，假设如下代码段
```c++
T a, b;
T c = a + b;
```
编译器可能会先对a+b的结果生成栈外临时对象，然后将该对象通过拷贝构造初始化c

更可能的实现是，直接将a+b的结果拷贝构造给c,没有栈外临时对象的创建。

如果编译器可以应用NRV优化，则可以直接在opeartor+()内部直接构造c。

那么编译器到底会采用哪种方法呢？c++标准对临时变量的生成没有规定，由编译器自行决定，只要保证，无论哪一种情况，最终c的结果都是正确的就可以了。

实现上，大部分编译器都会保证以下的表达式
```c++
T c = a + b;
```
其中加法操作，不管是这样定义的
```c++
T operator+( const T&, const T& );
```
or
```c++
T T::operator+( const T& );
```
都不会生成栈外的临时变量。


然而，如果是赋值表达式
```c++
c = a + b;
```
是会生成栈外临时变量的。


最后，如果表达式没有target:
```c++
a + b; // no target!
```
这种情况，必然会产生一个保存结果的临时对象，这种在存在subexpressions的表达式中尤为常见

比如，我们有如下变量
```c++
String s( "hello"), t( "world" ), u( "!" );
```
下述表达式
```c++
String v;
v = s + t + u;
```
或
```c++
printf( "%s\n", s + t );
```
results in a temporary being associated with the s + t subexpression.

临时对象的释放时机很重要，比如，如果`printf( "%s\n", s + t );`中，s+t subexpression temporary 的释放时机如下所示
```c++
// Pseudo C++ code: pre-Standard legal transformation
// temporary is destroyed too soon ...
String temp1 = operator+( s, t );
const char *temp2 = temp1.operator const char*();
// oops: legal but ill-advised (pre-Standard)
temp1.~String();
// undefined what temp2 is addressing at this point!
printf( "%s\n", temp2 );
```
因此，C++标准给临时对象的释放时机做了如下的规定
```
Temporary objects are destroyed as the last step in evaluating the **full-expression** that (lexically) contains the point where they were created. (Section 12.2)
```
What is a full-expression? Informally, it is the outermost containing expression. For example, consider the following:　（注意表达式和语句，也就是　expression 和 statement的区别，这里关注的是full-expression和sub-expression）
```c++
// tertiary full expression with 5 sub-expressions
(( objA > 1024 ) && ( objB > 1024 ))
? objA + objB : foo( objA, objB );
```
这个表达式中，子表达式有5个，full expresssion是最外面的`:?`表达式。在最外面的`:?`被计算完成之前，其中的4个子表达式的计算所产生的临时变量，都是不能被释放的。

这个规定，导致临时变量的释放，在条件表达式中的实现有点麻烦
```c++
if ( s + t || u + v )
{}
```
本来，u+v生成的临时变量，只有在其被计算时，才会产生，从而需要被释放。现在需要等到`||`表达式计算完成之后，才能统一释放。因此，在释放时，就需要某种机制，知道到底u+v有没有被计算。

也就是说，本来编译器可以做如下的转换
```c++
class X {
public:
  X();
  ~X();
  operator int();
  X foo();
private:
  int val;
};

main() {
X xx;
X yy;
if ( xx.foo() || yy.foo() );
return 0;
}
```
被转换成
```c++
int main (void ){
struct X __1xx ;
struct X __1yy ;
int __0_result;
// name_mangled default constructor:
// X::X( X *this )
__ct__1xFv ( & __1xx ) ;
__ct__1xFv ( & __1yy ) ;
{
// generated temporaries ...
struct X __0__Q1 ;
struct X __0__Q2 ;
int __0__Q3 ;
/* each side becomes a comma expression of
* the following sequence of steps:
*
* tempQ1 = xx.foo();
* tempQ3 = tempQ1.operator int();
* tempQ1.X::~X();
* tempQ3;
*/
// __opi__1xFv ==> X::operator int()
if ((((
__0__Q3 = __opi__1xFv(((
__0__Q1 = foo__1xFv( &__1xx )), (&__0__Q1 )))),
__dt__1xFv( &__0__Q1, 2) ), __0__Q3 )
|| (((
__0__Q3 = __opi__1xFv(((
__0__Q2 = foo__1xFv( & __1yy )), (&__0__Q2 )))),
__dt__1xFv( & __0__Q2, 2) ), __0__Q3 ));


{
{
__0_result = 0 ;
__dt__1xFv ( & __1yy , 2) ;
__dt__1xFv ( & __1xx , 2) ;
}
return __0_result ;
}

}

}
```

This strategy of placing the temporary's destructor in the evaluation of each subexpression circumvents the need to keep track of whether the second subexpression is actually evaluated. However, under the Standard's lifetime of temporaries rule, this implementation strategy is no longer permissible. The temporaries must not be destroyed until after evaluation of the full expression—that is, both sides—and so some form of conditional test must be inserted now to determine whether to destroy the temporary associated with the second subexpression.


临时对象的释放规则还就两个特殊情况做了补充，The first concerns an expression used to initialize an object; for example
```c++
bool verbose;
//...
String progNameVersion =
!verbose
? 0
: progName + progVersion;
```
`?:`表达式的计算会生成`progName + progVersion`子表达式的临时对象，然而这个对象不能在`?:`计算完成后就被释放，还要初始化`progNameVersion`呢!

因此标准做了如下的补充
```
…the temporary that holds the result of the expression shall persist until the object's initialization is complete.
```
但还是要注意如下的情况
```c++
// Oops: not a good idea
const char *progNameVersion = progName + progVersion;
```
progNameVersion 指向的内存是临时对象的内存，这个内存在该表达式之后就被释放了，其转换如下
```c++
// Pseudo C++ Code
String temp;
operator+( temp, progName, progVersion );
progNameVersion = temp.String::operator char*();
temp.String::~String();
```
当然这种错误就应该算是程序员使用的错误了。


The second exception to the lifetime of temporaries rule concerns when a temporary is bound to a reference. For example,
```c++
const String &space = " ";
```
generates code that looks something like
```c++
// Pseudo C++ Code
String temp;
temp.String::String( " " );
const String &space = temp;
```
Obviously, if the temporary were destroyed now, the reference would be slightly less than useless. So the rule is that a temporary bound to a reference persists for the lifetime of the reference or until the end of the scope in which the temporary is created, whichever comes first.
