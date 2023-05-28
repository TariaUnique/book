![](../../../images/c%26c%2B%2B/stl01.png)

![](../../../images/c%26c%2B%2B/stl02.png)

容器使用分配器分配内存。
算法使用迭代器操作容器的元素。
算法使用仿函数辅助操作，这个辅助操作通常被称为predicate(谓词，断言.....)

适配器分为三种：
* 容器适配器，比如stack,queue
* 迭代器适配器, 比如reverse_iterator, inserter
* 仿函数适配器,比如bind,not1

![](../../../images/c%26c%2B%2B/stl03.png)


默认使用的分配器：std::allocator
G4.9
```c++
  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class vector : protected _Vector_base<_Tp, _Alloc>


  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class list : protected _List_base<_Tp, _Alloc>

  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class deque : protected _Deque_base<_Tp, _Alloc>

  template<typename _Key, typename _Compare = std::less<_Key>,
	   typename _Alloc = std::allocator<_Key> >
    class set

  template <typename _Key, typename _Tp, typename _Compare = std::less<_Key>,
            typename _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
    class map

  template<class _Value,
	   class _Hash = hash<_Value>,
	   class _Pred = std::equal_to<_Value>,
	   class _Alloc = std::allocator<_Value> >
    class unordered_set

  template<class _Key, class _Tp,
	   class _Hash = hash<_Key>,
	   class _Pred = std::equal_to<_Key>,
	   class _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
    class unordered_map
```

std::allocator的类型如下,继承自__allocator_base
```c++
  template<typename _Tp>
    class allocator: public __allocator_base<_Tp>
```
__allocator_base是个naming alias,默认类型如下
```c++
  /**
   *  @brief  An alias to the base class for std::allocator.
   *  @ingroup allocators
   *
   *  Used to set the std::allocator base class to
   *  __gnu_cxx::new_allocator.
   *
   *  @tparam  _Tp  Type of allocated object.
    */
  template<typename _Tp>
    using __allocator_base = __gnu_cxx::new_allocator<_Tp>;
```
也就是实际使用的分配器是new_allocator,new_allocator在分配内存和释放内存时，直接使用operator new 和operator delete

除此之外，G4.9还提供了其他的分配器，比如`__gnu_cxx::bitmap_allocator`,`__gnu_cxx::__pool_alloc`,`__gnu_cxx::__mt_alloc`,`__gnu_cxx::malloc_allocator`

如果需要使用别的分配器，代码如下
![](../../../images/c%26c%2B%2B/stl04.png)
![](../../../images/c%26c%2B%2B/stl05.png)

模板三大类：类模板，函数模板，成员模板
![](../../../images/c%26c%2B%2B/stl06.png)
![](../../../images/c%26c%2B%2B/stl07.png)
![](../../../images/c%26c%2B%2B/stl08.png)
注意上图中的`#ifdef __STL_MEMBER_TEMPLATES`现在已经没有必要加上了，modern c++ compliers all support member templates
```c++
#include <iostream>

class MyClass {
public:
    template<typename T>
    void print(const T& value) {
        std::cout << "Value: " << value << std::endl;
    }
};

int main() {
    MyClass obj;
    obj.print<int>(42);
    obj.print<std::string>("Hello, world!");

    return 0;
}
```
类模板有特化和偏特化

![](../../../images/c%26c%2B%2B/stl09.png)
这是G2.9实现的__type_traits，现在不需要使用__type_traits了。

![](../../../images/c%26c%2B%2B/stl10.png)


![](../../../images/c%26c%2B%2B/stl11.png)
偏特化有两种，一种是对模板参数的数量的偏特化，只提供部分模板参数的实参类型。第二种是对模板参数的类型的偏特化。

函数模板的模板实参是编译器推导的，没有所谓的特化的概念，或者说，没有偏特化。函数模板的特化就是重载。


operator new 和operator delete
operator new 和 operator delete 在c++的全局空间中，使用时需要加上`::`

G4.9 operator new
```c++
_GLIBCXX_WEAK_DEFINITION void *
operator new (std::size_t sz) _GLIBCXX_THROW (std::bad_alloc)
{
  void *p;

  /* malloc (0) is unpredictable; avoid it.  */
  if (sz == 0)
    sz = 1;
  p = (void *) malloc (sz);
  while (p == 0)
    {
      new_handler handler = std::get_new_handler ();
      if (! handler)
	_GLIBCXX_THROW_OR_ABORT(bad_alloc());
      handler ();
      p = (void *) malloc (sz);
    }

  return p;
}
```

operator delete 
```c++
_GLIBCXX_WEAK_DEFINITION void
operator delete(void* ptr) _GLIBCXX_USE_NOEXCEPT
{
  std::free(ptr);
}
```

关于这个`_GLIBCXX_WEAK_DEFINITION`,GPT explain as follows:

_GLIBCXX_WEAK_DEFINITION is a macro used in the GNU C++ Standard Library (libstdc++) to mark a function or an object to have weak linkage. Weak linkage means that if a symbol is defined multiple times, the linker will choose one of the available definitions, rather than report a violation of the One Definition Rule (ODR). This is mainly used for inline functions and templates to handle possible multiple definitions arising from their inclusion in different translation units.

所以，这个函数是可以被别的库重新定义的，比如ptmalloc中就重新定义了operator delete 和operator new, 使用的时候，只需要链接ptmalloc就可以了。


之前说过，G4.9版本的stl中，所有容器的分配器都是`__gnu_cxx::new_allocator`,看下其实现
```c++
 template<typename _Tp>
    class new_allocator
    {
    public:
      typedef size_t     size_type;
      typedef ptrdiff_t  difference_type;
      typedef _Tp*       pointer;
      typedef const _Tp* const_pointer;
      typedef _Tp&       reference;
      typedef const _Tp& const_reference;
      typedef _Tp        value_type;

      template<typename _Tp1>
        struct rebind
        { typedef new_allocator<_Tp1> other; }; 

//....


      // NB: __n is permitted to be 0.  The C++ standard says nothing
      // about what the return value is when __n == 0.
      pointer
      allocate(size_type __n, const void* = 0)
      { 
     	if (__n > this->max_size())
     	  std::__throw_bad_alloc();

     	return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp)));
      }

      // __p is not permitted to be a null pointer.
      void
      deallocate(pointer __p, size_type)
      { ::operator delete(__p); }

      size_type
      max_size() const _GLIBCXX_USE_NOEXCEPT
      { return size_t(-1) / sizeof(_Tp); }

    #if __cplusplus >= 201103L
      template<typename _Up, typename... _Args>
        void construct(_Up* __p, _Args&&... __args){ 
            ::new((void *)__p) _Up(std::forward<_Args>(__args)...); 
        }

      template<typename _Up>
        void destroy(_Up* __p) { __p->~_Up(); }

    };

```
所有类型的allocator,都需要提供allocate,deallocate,construct,destroy四个方法。

G2.9版本的stl中，默认的分配器是alloc,alloc使用链表缓存分配的内存块。
![](../../../images/c%26c%2B%2B/stl12.png)
侯捷说，G4.9中的`__gnu_cxx::__pool_alloc`就是G2.9中的alloc,那我们看下`__gnu_cxx::__pool_alloc`的实现
```c++
  template<typename _Tp>
    class __pool_alloc : private __pool_alloc_base
    {
        //...
    public:
      typedef size_t     size_type;
      typedef ptrdiff_t  difference_type;
      typedef _Tp*       pointer;
      typedef const _Tp* const_pointer;
      typedef _Tp&       reference;
      typedef const _Tp& const_reference;
      typedef _Tp        value_type;

      template<typename _Tp1>
        struct rebind
        { typedef __pool_alloc<_Tp1> other; };
    //...


      template<typename _Up, typename... _Args>
        void construct(_Up* __p, _Args&&... __args){ 
            ::new((void *)__p) _Up(std::forward<_Args>(__args)...); 
        }

      template<typename _Up>
        void destroy(_Up* __p) { __p->~_Up(); }

      pointer allocate(size_type __n, const void* = 0);

      void deallocate(pointer __p, size_type __n);      
    };


  template<typename _Tp>
    _Tp* __pool_alloc<_Tp>::allocate(size_type __n, const void*)
    {
        pointer __ret = 0;
        if (__builtin_expect(__n != 0, true))
      	{
            //...

      	    const size_t __bytes = __n * sizeof(_Tp);	      
      	    if (__bytes > size_t(_S_max_bytes) || _S_force_new > 0)
      	        __ret = static_cast<_Tp*>(::operator new(__bytes));
      	    else {
         	    _Obj* volatile* __free_list = _M_get_free_list(__bytes);
         	      
         	    __scoped_lock sentry(_M_get_mutex());
         	    _Obj* __restrict__ __result = *__free_list;
         	    if (__builtin_expect(__result == 0, 0))
         		  __ret = static_cast<_Tp*>(_M_refill(_M_round_up(__bytes)));
         	    else {
         		   *__free_list = __result->_M_free_list_link;
         		   __ret = reinterpret_cast<_Tp*>(__result);
         		}
         	    if (__ret == 0)
         		  std::__throw_bad_alloc();
      	    }
      	}
        return __ret;
    }

  template<typename _Tp>
    void __pool_alloc<_Tp>::deallocate(pointer __p, size_type __n)
    {
        if (__builtin_expect(__n != 0 && __p != 0, true))
       {
            const size_t __bytes = __n * sizeof(_Tp);
            if (__bytes > static_cast<size_t>(_S_max_bytes) || _S_force_new > 0)
                ::operator delete(__p);
            else
            {
               _Obj* volatile* __free_list = _M_get_free_list(__bytes);
               _Obj* __q = reinterpret_cast<_Obj*>(__p);

               __scoped_lock sentry(_M_get_mutex());
               __q ->_M_free_list_link = *__free_list;
               *__free_list = __q;
            }
       }
    }



    class __pool_alloc_base
    {
    protected:

      enum { _S_align = 8 };
      enum { _S_max_bytes = 128 };
      enum { _S_free_list_size = (size_t)_S_max_bytes / (size_t)_S_align };
      
      union _Obj
      {
	    union _Obj* _M_free_list_link;
	    char        _M_client_data[1];    // The client sees this. 
      };
      
      static _Obj* volatile         _S_free_list[_S_free_list_size];
      //....
    };
```

容器分为序列式容器和关联式容器
* 序列式容器：array,vector, list, forward_list, deque
* 关联式容器：map, set, unordered_map,unordered_set,multimap,multiset, unordered_multimap,unordered_multiset



list实现 G2.8
```c++

template <class T>
struct __list_node {
  typedef void* void_pointer;
  void_pointer next;  //void* 
  void_pointer prev;  //void* 
  T data;
};


template <class T, class Alloc = alloc>
class list {
    //....
protected:
  link_type node;   //一个指针node的指针  typedef list_node* link_type;  typedef __list_node<T> list_node;

public:
  typedef __list_iterator<T, T&, T*>             iterator;
//...


protected:
  typedef simple_alloc<list_node, Alloc> list_node_allocator;
  link_type get_node() { return list_node_allocator::allocate(); }
  void put_node(link_type p) { list_node_allocator::deallocate(p); }

  link_type create_node(const T& x) {
    link_type p = get_node();
    __STL_TRY {
      construct(&p->data, x);
    }
    __STL_UNWIND(put_node(p));
    return p;
  }
  void destroy_node(link_type p) {
    destroy(&p->data);
    put_node(p);
  }

protected:
  void empty_initialize() { 
    node = get_node();
    node->next = node;
    node->prev = node;
  }

//...


public:
//...
  list() { empty_initialize(); }

  iterator begin() { return (link_type)((*node).next); }

  iterator end() { return node; }

  
  reference front() { return *begin(); }
  reference back() { return *(--end()); }




  void push_front(const T& x) { insert(begin(), x); }
  void push_back(const T& x) { insert(end(), x); }
  iterator erase(iterator position) {
    link_type next_node = link_type(position.node->next);
    link_type prev_node = link_type(position.node->prev);
    prev_node->next = next_node;
    next_node->prev = prev_node;
    destroy_node(position.node);
    return iterator(next_node);
  }
  void pop_front() { erase(begin()); }
  void pop_back() { 
    iterator tmp = end();
    erase(--tmp);
  }
 
public:
//...
  void remove(const T& value);
  void unique();
  void merge(list& x);
  void reverse();
  void sort();

  template <class Predicate> void remove_if(Predicate);
  template <class BinaryPredicate> void unique(BinaryPredicate);
  template <class StrictWeakOrdering> void merge(list&, StrictWeakOrdering);
  template <class StrictWeakOrdering> void sort(StrictWeakOrdering);

};



```

list迭代器  ` __list_iterator`
```c++
template<class T, class Ref, class Ptr>
struct __list_iterator {
  typedef __list_iterator<T, T&, T*>             iterator;
  typedef __list_iterator<T, const T&, const T*> const_iterator;
  typedef __list_iterator<T, Ref, Ptr>           self;

  typedef bidirectional_iterator_tag iterator_category;
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  typedef __list_node<T>* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;

  link_type node;

  __list_iterator(link_type x) : node(x) {}
  __list_iterator() {}
  __list_iterator(const iterator& x) : node(x.node) {}

  bool operator==(const self& x) const { return node == x.node; }
  bool operator!=(const self& x) const { return node != x.node; }
  reference operator*() const { return (*node).data; }

  pointer operator->() const { return &(operator*()); }


  self& operator++() { 
    node = (link_type)((*node).next);
    return *this;
  }
  self operator++(int) { 
    self tmp = *this;
    ++*this;
    return tmp;
  }
  self& operator--() { 
    node = (link_type)((*node).prev);
    return *this;
  }
  self operator--(int) { 
    self tmp = *this;
    --*this;
    return tmp;
  }
};
```
![](../../../images/c%26c%2B%2B/stl14.png)
![](../../../images/c%26c%2B%2B/stl15.png)
![](../../../images/c%26c%2B%2B/stl16.png)
![](../../../images/c%26c%2B%2B/stl17.png)

所有的迭代器都需要提供5种迭代器的属性，也就是5个typedef: iterator_category,value_type,pointer,reference,difference_type

这些迭代器属性可以通过iterator_traits萃取机来获取
```c++
//G2.8
template <class Iterator>
struct iterator_traits {
  typedef typename Iterator::iterator_category iterator_category;
  typedef typename Iterator::value_type        value_type;
  typedef typename Iterator::difference_type   difference_type;
  typedef typename Iterator::pointer           pointer;
  typedef typename Iterator::reference         reference;
};

```

比如获取迭代器的iterator_category属性，可以写一个如下的函数模板
```c++
template <typename Iter>
inline typename iterator_traits<Iter>::iterator_category 
_iterator_category(const Iter& I){
    return typename iterator_traits<Iter>::iterator_category();
}
```
任何一种iterator_category都是空class,这里返回该空class的对象，大小是1。

通常直接如下获取迭代器属性即可
```c++
typedef typename iterator_traits<Iter>::value_type ValueType;
```

通常迭代器是个class，其中定义了这5个属性。所以，算法在操作迭代器时，应该可以直接从迭代器中获取这5个属性即可，为什么还需要用iterator_traits呢？
![](../../../images/c%26c%2B%2B/stl18.png)
答案是，有的迭代器就是简单的指针，比如vector的迭代器，就是T* 指针
```c++
//G2.8
template <class T, class Alloc = alloc>
class vector {
public:
  typedef T value_type;
  typedef value_type* pointer;
  typedef value_type* iterator;
//...
}
```
这时候，就可以使用类模板的偏特化，提供iterator_traits的模板参数为指针的实现
```c++
template <class T>
struct iterator_traits<T*> {
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef T*                         pointer;
  typedef T&                         reference;
};

template <class T>
struct iterator_traits<const T*> {
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef const T*                   pointer;
  typedef const T&                   reference;
};

```
注意指针的iterator_category是random_access_iterator_tag


这样，不管实际的iterator是个类还是个指针，iterator_traits都可以获取iterator的5个属性
![](../../../images/c%26c%2B%2B/stl19.png)
![](../../../images/c%26c%2B%2B/stl20.png)

vector G2.8实现
```c++
template <class T, class Alloc = alloc>
class vector {
public:
  typedef T value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type* iterator;
  typedef const value_type* const_iterator;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;

//....

protected:
  typedef simple_alloc<value_type, Alloc> data_allocator;
  iterator start;
  iterator finish;
  iterator end_of_storage;
  void insert_aux(iterator position, const T& x);
  void deallocate() {
    if (start) data_allocator::deallocate(start, end_of_storage - start);
  }

  void fill_initialize(size_type n, const T& value) {
    start = allocate_and_fill(n, value);
    finish = start + n;
    end_of_storage = finish;
  }
public:
  iterator begin() { return start; }
  const_iterator begin() const { return start; }
  iterator end() { return finish; }
  const_iterator end() const { return finish; }
  //....
  size_type size() const { return size_type(end() - begin()); }
  size_type max_size() const { return size_type(-1) / sizeof(T); }
  size_type capacity() const { return size_type(end_of_storage - begin()); }
  bool empty() const { return begin() == end(); }
  reference operator[](size_type n) { return *(begin() + n); }
  const_reference operator[](size_type n) const { return *(begin() + n); }

  vector() : start(0), finish(0), end_of_storage(0) {}
  vector(size_type n, const T& value) { fill_initialize(n, value); }
//....
  explicit vector(size_type n) { fill_initialize(n, T()); }

  vector(const vector<T, Alloc>& x) {
    start = allocate_and_copy(x.end() - x.begin(), x.begin(), x.end());
    finish = start + (x.end() - x.begin());
    end_of_storage = finish;
  }

  template <class InputIterator>
  vector(InputIterator first, InputIterator last) :
    start(0), finish(0), end_of_storage(0)
  {
    range_initialize(first, last, iterator_category(first));
  }
  //...

  ~vector() { 
    destroy(start, finish);
    deallocate();
  }
  vector<T, Alloc>& operator=(const vector<T, Alloc>& x);
//...
  reference front() { return *begin(); }
  const_reference front() const { return *begin(); }
  reference back() { return *(end() - 1); }
  const_reference back() const { return *(end() - 1); }
  void push_back(const T& x) {
    if (finish != end_of_storage) {
      construct(finish, x);
      ++finish;
    }
    else
      insert_aux(end(), x);
  }
  void swap(vector<T, Alloc>& x) {
    __STD::swap(start, x.start);
    __STD::swap(finish, x.finish);
    __STD::swap(end_of_storage, x.end_of_storage);
  }
  iterator insert(iterator position, const T& x) {
    size_type n = position - begin();
    if (finish != end_of_storage && position == end()) {
      construct(finish, x);
      ++finish;
    }
    else
      insert_aux(position, x);
    return begin() + n;
  }
//...
  void pop_back() {
    --finish;
    destroy(finish);
  }
  iterator erase(iterator position) {
    if (position + 1 != end())
      copy(position + 1, finish, position);
    --finish;
    destroy(finish);
    return position;
  }
  iterator erase(iterator first, iterator last) {
    iterator i = copy(last, finish, first);
    destroy(i, finish);
    finish = finish - (last - first);
    return first;
  }
  void resize(size_type new_size, const T& x) {
    if (new_size < size()) 
      erase(begin() + new_size, end());
    else
      insert(end(), new_size - size(), x);
  }
  void resize(size_type new_size) { resize(new_size, T()); }
  void clear() { erase(begin(), end()); }

protected:
  iterator allocate_and_fill(size_type n, const T& x) {
    iterator result = data_allocator::allocate(n);
    try{
      uninitialized_fill_n(result, n, x);
      return result;
    }catch(...){
      data_allocator::deallocate(result, n);
      throw;
    }
  }

  template <class ForwardIterator>
  iterator allocate_and_copy(size_type n,
                             ForwardIterator first, ForwardIterator last) {
    iterator result = data_allocator::allocate(n);
    __STL_TRY {
      uninitialized_copy(first, last, result);
      return result;
    }
    __STL_UNWIND(data_allocator::deallocate(result, n));
  }

//...

  template <class InputIterator>
  void range_initialize(InputIterator first, InputIterator last,
                        input_iterator_tag) {
    for ( ; first != last; ++first)
      push_back(*first);
  }

  // This function is only called by the constructor.  We have to worry
  //  about resource leaks, but not about maintaining invariants.
  template <class ForwardIterator>
  void range_initialize(ForwardIterator first, ForwardIterator last,
                        forward_iterator_tag) {
    size_type n = 0;
    distance(first, last, n);
    start = allocate_and_copy(n, first, last);
    finish = start + n;
    end_of_storage = finish;
  }

//...
};

template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x) {
  if (finish != end_of_storage) {
    construct(finish, *(finish - 1));
    ++finish;
    T x_copy = x;
    copy_backward(position, finish - 2, finish - 1);
    *position = x_copy;
  }
  else {
    const size_type old_size = size();
    const size_type len = old_size != 0 ? 2 * old_size : 1;
    iterator new_start = data_allocator::allocate(len);
    iterator new_finish = new_start;
    try{
      new_finish = uninitialized_copy(start, position, new_start);
      construct(new_finish, x);
      ++new_finish;
      new_finish = uninitialized_copy(position, finish, new_finish);
    }
    catch(...) {
      destroy(new_start, new_finish); 
      data_allocator::deallocate(new_start, len);
      throw;
    }
    destroy(begin(), end());
    deallocate();
    start = new_start;
    finish = new_finish;
    end_of_storage = new_start + len;
  }
}


template <class BidirectionalIterator1, class BidirectionalIterator2>
inline BidirectionalIterator2 __copy_backward(BidirectionalIterator1 first, 
                                              BidirectionalIterator1 last, 
                                              BidirectionalIterator2 result) {
  while (first != last) *--result = *--last;
  return result;
}

```
重点
![](../../../images/c%26c%2B%2B/stl22.png)
![](../../../images/c%26c%2B%2B/stl23.png)
![](../../../images/c%26c%2B%2B/stl24.png)

vector的迭代器就是指针，在被iterator_traits萃取时，使用的是iterator_traits对指针偏特化的模板
```c++
template <class T>
struct iterator_traits<T*> {
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef T*                         pointer;
  typedef T&                         reference;
};
```

比如如下代码
```c++
typedef typename iterator_traits<vector<int>::iterator>::value_type ValueType;
```
ValueType 为 int;


![](../../../images/c%26c%2B%2B/stl26.png)
![](../../../images/c%26c%2B%2B/stl27.png)

forward_list:
```c++
  template<typename _Tp, typename _Alloc = allocator<_Tp> >
    class forward_list : private _Fwd_list_base<_Tp, _Alloc>{
      //...
    };

 template<typename _Tp, typename _Alloc>
    struct _Fwd_list_base
    {
      //....
    protected:
      struct _Fwd_list_impl 
      : public _Node_alloc_type
      {
        _Fwd_list_node_base _M_head;
        //....
      };

      _Fwd_list_impl _M_impl;
    };

  struct _Fwd_list_node_base {
    //...
    _Fwd_list_node_base* _M_next;
  };

  template<typename _Tp>
    struct _Fwd_list_node
    : public _Fwd_list_node_base
    {
      //...
      __gnu_cxx::__aligned_buffer<_Tp> _M_storage;

    };

```
![](../../../images/c%26c%2B%2B/stl28.png)


deque G2.9实现
![](../../../images/c%26c%2B%2B/stl72.png)
![](../../../images/c%26c%2B%2B/stl73.png)
![](../../../images/c%26c%2B%2B/stl74.png)
![](../../../images/c%26c%2B%2B/stl75.png)
insert_aux的实现
```c++
template <class T, class Alloc, size_t BufSize>
typename deque<T, Alloc, BufSize>::iterator
deque<T, Alloc, BufSize>::insert_aux(iterator pos, const value_type& x) {
  difference_type index = pos - start;
  value_type x_copy = x;
  if (index < size() / 2) {
    push_front(front());
    iterator front1 = start;
    ++front1;
    iterator front2 = front1;
    ++front2;
    pos = start + index;
    iterator pos1 = pos;
    ++pos1;
    copy(front2, pos1, front1);
  }
  else {
    push_back(back());
    iterator back1 = finish;
    --back1;
    iterator back2 = back1;
    --back2;
    pos = start + index;
    copy_backward(pos, back2, back1);
  }
  *pos = x_copy;
  return pos;
}
```
![](../../../images/c%26c%2B%2B/stl76.png)
![](../../../images/c%26c%2B%2B/stl77.png)
![](../../../images/c%26c%2B%2B/stl78.png)
![](../../../images/c%26c%2B%2B/stl79.png)


deque G4.9实现
![](../../../images/c%26c%2B%2B/stl64.png)
![](../../../images/c%26c%2B%2B/stl80.png)


容器适配器：queue， stack
![](../../../images/c%26c%2B%2B/stl81.png)
![](../../../images/c%26c%2B%2B/stl82.png)
![](../../../images/c%26c%2B%2B/stl83.png)
![](../../../images/c%26c%2B%2B/stl84.png)
![](../../../images/c%26c%2B%2B/stl85.png)


![](../../../images/c%26c%2B%2B/stl86.png)
![](../../../images/c%26c%2B%2B/stl87.png)
![](../../../images/c%26c%2B%2B/stl88.png)
![](../../../images/c%26c%2B%2B/stl89.png)

G2.8实现
```c++
typedef bool __rb_tree_color_type;
const __rb_tree_color_type __rb_tree_red = false;
const __rb_tree_color_type __rb_tree_black = true;

struct __rb_tree_node_base
{
  typedef __rb_tree_color_type color_type;
  typedef __rb_tree_node_base* base_ptr;

  color_type color; 
  base_ptr parent;
  base_ptr left;
  base_ptr right;

  static base_ptr minimum(base_ptr x)
  {
    while (x->left != 0) x = x->left;
    return x;
  }

  static base_ptr maximum(base_ptr x)
  {
    while (x->right != 0) x = x->right;
    return x;
  }
};

template <class Value>
struct __rb_tree_node : public __rb_tree_node_base
{
  typedef __rb_tree_node<Value>* link_type;
  Value value_field;
};


template <class Key, class Value, class KeyOfValue, class Compare,
          class Alloc = alloc>
class rb_tree {
//....
protected:
  typedef void* void_pointer;
  typedef __rb_tree_node_base* base_ptr;
  typedef __rb_tree_node<Value> rb_tree_node;
  typedef simple_alloc<rb_tree_node, Alloc> rb_tree_node_allocator;
  typedef __rb_tree_color_type color_type;
public:
  typedef Key key_type;
  typedef Value value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef rb_tree_node* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;

protected:
  size_type node_count; // keeps track of size of tree
  link_type header;  
  Compare key_compare;

  link_type& root() const { return (link_type&) header->parent; }
  link_type& leftmost() const { return (link_type&) header->left; }
  link_type& rightmost() const { return (link_type&) header->right; }


  static reference value(link_type x) { return x->value_field; }
  static const Key& key(link_type x) { return KeyOfValue()(value(x)); }
  static color_type& color(link_type x) { return (color_type&)(x->color); }




public:
  typedef __rb_tree_iterator<value_type, reference, pointer> iterator;


public:
                                // allocation/deallocation
  rb_tree(const Compare& comp = Compare())
    : node_count(0), key_compare(comp) { init(); }

  void init() {
    header = get_node();
    color(header) = __rb_tree_red; // used to distinguish header from 
                                   // root, in iterator.operator++
    root() = 0;
    leftmost() = header;
    rightmost() = header;
  }



public:    
  iterator begin() { return leftmost(); }
  iterator end() { return header; }
    
public:
                                // insert/erase
  pair<iterator,bool> insert_unique(const value_type& x);
  iterator insert_equal(const value_type& x);

  void erase(iterator position);
  size_type erase(const key_type& x);
  void erase(iterator first, iterator last);
  void erase(const key_type* first, const key_type* last);

public:
                                // set operations:
  iterator find(const key_type& x);
  size_type count(const key_type& x) const;

};

//rb_tree的迭代器
struct __rb_tree_base_iterator
{
  typedef __rb_tree_node_base::base_ptr base_ptr;
  typedef bidirectional_iterator_tag iterator_category;
  typedef ptrdiff_t difference_type;
  base_ptr node;

  void increment()
  {
    if (node->right != 0) {
      node = node->right;
      while (node->left != 0)
        node = node->left;
    }
    else {
      base_ptr y = node->parent;  
      while (node == y->right) {
        node = y;
        y = y->parent;
      }
      if (node->right != y) 
        node = y;
    }
  }

  void decrement()
  {
    if (node->color == __rb_tree_red &&
        node->parent->parent == node)
      node = node->right;
    else if (node->left != 0) {
      base_ptr y = node->left;
      while (y->right != 0)
        y = y->right;
      node = y;
    }
    else {
      base_ptr y = node->parent;
      while (node == y->left) {
        node = y;
        y = y->parent;
      }
      node = y;
    }
  }
};

template <class Value, class Ref, class Ptr>
struct __rb_tree_iterator : public __rb_tree_base_iterator
{
  typedef Value value_type;
  typedef Ref reference;
  typedef Ptr pointer;
  typedef __rb_tree_iterator<Value, Value&, Value*>             iterator;
  typedef __rb_tree_iterator<Value, const Value&, const Value*> const_iterator;
  typedef __rb_tree_iterator<Value, Ref, Ptr>                   self;
  typedef __rb_tree_node<Value>* link_type;

  __rb_tree_iterator() {}
  __rb_tree_iterator(link_type x) { node = x; }
  __rb_tree_iterator(const iterator& it) { node = it.node; }

  reference operator*() const { return link_type(node)->value_field; }
  pointer operator->() const { return &(operator*()); }

  self& operator++() { increment(); return *this; }
  self operator++(int) {
    self tmp = *this;
    increment();
    return tmp;
  }
    
  self& operator--() { decrement(); return *this; }
  self operator--(int) {
    self tmp = *this;
    decrement();
    return tmp;
  }
};

```

![](../../../images/c%26c%2B%2B/stl91.png)

![](../../../images/c%26c%2B%2B/stl92.png)

![](../../../images/c%26c%2B%2B/stl93.png)


![](../../../images/c%26c%2B%2B/stl94.png)



![](../../../images/c%26c%2B%2B/stl95.png)

![](../../../images/c%26c%2B%2B/stl96.png)

![](../../../images/c%26c%2B%2B/stl97.png)


![](../../../images/c%26c%2B%2B/stl98.png)

![](../../../images/c%26c%2B%2B/stl99.png)

![](../../../images/c%26c%2B%2B/stl100.png)

hashtable G2.8实现
```c++

template <class Value>
struct __hashtable_node
{
  __hashtable_node* next;
  Value val;
};  


template <class Value, class Key, class HashFcn,
          class ExtractKey, class EqualKey,
          class Alloc>
class hashtable {
  //...
public:
  typedef Key key_type;
  typedef Value value_type;
  typedef HashFcn hasher;
  typedef EqualKey key_equal;

  typedef size_t            size_type;
  typedef ptrdiff_t         difference_type;
  typedef value_type*       pointer;
  typedef const value_type* const_pointer;
  typedef value_type&       reference;
  typedef const value_type& const_reference;

  hasher hash_funct() const { return hash; }
  key_equal key_eq() const { return equals; }

private:
  hasher hash;
  key_equal equals;
  ExtractKey get_key;

  typedef __hashtable_node<Value> node;
  typedef simple_alloc<node, Alloc> node_allocator;

  vector<node*,Alloc> buckets;
  size_type num_elements;

public:
  typedef __hashtable_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc> iterator;



public:
  hashtable(size_type n,
            const HashFcn&    hf,
            const EqualKey&   eql,
            const ExtractKey& ext)
    : hash(hf), equals(eql), get_key(ext), num_elements(0)
  {
    initialize_buckets(n);
  }

  hashtable(size_type n,
            const HashFcn&    hf,
            const EqualKey&   eql)
    : hash(hf), equals(eql), get_key(ExtractKey()), num_elements(0)
  {
    initialize_buckets(n);
  }


  ~hashtable() { clear(); }

  size_type size() const { return num_elements; }
  bool empty() const { return size() == 0; }



  iterator begin()
  { 
    for (size_type n = 0; n < buckets.size(); ++n)
      if (buckets[n])
        return iterator(buckets[n], this);
    return end();
  }

  iterator end() { return iterator(0, this); }


public:

  size_type bucket_count() const { return buckets.size(); }

  size_type max_bucket_count() const
    { return __stl_prime_list[__stl_num_primes - 1]; } 

  size_type elems_in_bucket(size_type bucket) const
  {
    size_type result = 0;
    for (node* cur = buckets[bucket]; cur; cur = cur->next)
      result += 1;
    return result;
  }

  pair<iterator, bool> insert_unique(const value_type& obj)
  {
    resize(num_elements + 1);
    return insert_unique_noresize(obj);
  }

  iterator insert_equal(const value_type& obj)
  {
    resize(num_elements + 1);
    return insert_equal_noresize(obj);
  }

  pair<iterator, bool> insert_unique_noresize(const value_type& obj);
  iterator insert_equal_noresize(const value_type& obj);
 


  iterator find(const key_type& key) 
  {
    size_type n = bkt_num_key(key);
    node* first;
    for ( first = buckets[n];
          first && !equals(get_key(first->val), key);
          first = first->next)
      {}
    return iterator(first, this);
  } 


  size_type count(const key_type& key) const
  {
    const size_type n = bkt_num_key(key);
    size_type result = 0;

    for (const node* cur = buckets[n]; cur; cur = cur->next)
      if (equals(get_key(cur->val), key))
        ++result;
    return result;
  }

  size_type erase(const key_type& key);
  void erase(const iterator& it);
  void erase(iterator first, iterator last);

  void resize(size_type num_elements_hint);
  void clear();

private:
  size_type next_size(size_type n) const { return __stl_next_prime(n); }

  void initialize_buckets(size_type n)
  {
    const size_type n_buckets = next_size(n);
    buckets.reserve(n_buckets);
    buckets.insert(buckets.end(), n_buckets, (node*) 0);
    num_elements = 0;
  }

  size_type bkt_num_key(const key_type& key, size_t n) const
  {
    return hash(key) % n;
  }

};

//iterator

template <class Value, class Key, class HashFcn,
          class ExtractKey, class EqualKey, class Alloc>
struct __hashtable_iterator {
  typedef hashtable<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc>
          hashtable;
  typedef __hashtable_iterator<Value, Key, HashFcn, 
                               ExtractKey, EqualKey, Alloc>
          iterator;
  typedef __hashtable_const_iterator<Value, Key, HashFcn, 
                                     ExtractKey, EqualKey, Alloc>
          const_iterator;
  typedef __hashtable_node<Value> node;

  typedef forward_iterator_tag iterator_category;
  typedef Value value_type;
  typedef ptrdiff_t difference_type;
  typedef size_t size_type;
  typedef Value& reference;
  typedef Value* pointer;

  node* cur;
  hashtable* ht;
  __hashtable_iterator(node* n, hashtable* tab) : cur(n), ht(tab) {}
  __hashtable_iterator() {}
  reference operator*() const { return cur->val; }
  pointer operator->() const { return &(operator*()); }
  iterator& operator++();
  iterator operator++(int);
  bool operator==(const iterator& it) const { return cur == it.cur; }
  bool operator!=(const iterator& it) const { return cur != it.cur; }
};

template <class V, class K, class HF, class ExK, class EqK, class A>
__hashtable_iterator<V, K, HF, ExK, EqK, A>&
__hashtable_iterator<V, K, HF, ExK, EqK, A>::operator++()
{
  const node* old = cur;
  cur = cur->next;
  if (!cur) {
    size_type bucket = ht->bkt_num(old->val);
    while (!cur && ++bucket < ht->buckets.size())
      cur = ht->buckets[bucket];
  }
  return *this;
}

```
hashtable的iterator在从begin()遍历到end(),元素并不是有序的哈！


stl中提供了几个默认的哈希函数
```c++
template <class Key> struct hash { };

inline size_t __stl_hash_string(const char* s)
{
  unsigned long h = 0; 
  for ( ; *s; ++s)
    h = 5*h + *s;
  
  return size_t(h);
}
//__STL_TEMPLATE_NULL 为 template<>
__STL_TEMPLATE_NULL struct hash<char*>
{
  size_t operator()(const char* s) const { return __stl_hash_string(s); }
};

__STL_TEMPLATE_NULL struct hash<const char*>
{
  size_t operator()(const char* s) const { return __stl_hash_string(s); }
};

__STL_TEMPLATE_NULL struct hash<char> {
  size_t operator()(char x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<unsigned char> {
  size_t operator()(unsigned char x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<signed char> {
  size_t operator()(unsigned char x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<short> {
  size_t operator()(short x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<unsigned short> {
  size_t operator()(unsigned short x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<int> {
  size_t operator()(int x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<unsigned int> {
  size_t operator()(unsigned int x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<long> {
  size_t operator()(long x) const { return x; }
};
__STL_TEMPLATE_NULL struct hash<unsigned long> {
  size_t operator()(unsigned long x) const { return x; }
};

```

G2.9没有提供hash<std::string>,G4.9中定义如下
```c++
  template<>
    struct hash<string>
    : public __hash_base<size_t, string>
    {
      size_t
      operator()(const string& __s) const noexcept
      { return std::_Hash_impl::hash(__s.data(), __s.length()); }
    };
```

![](../../../images/c%26c%2B%2B/stl101.png)

unordered_set,unordered_map,unordered_multiset, unordered_multimap在G4.9中的定义如下
```c++
    template<class _Value,
	   class _Hash = hash<_Value>,
	   class _Pred = std::equal_to<_Value>,
	   class _Alloc = std::allocator<_Value> >
    class unordered_set
  
    template<class _Key, class _Tp,
	   class _Hash = hash<_Key>,
	   class _Pred = std::equal_to<_Key>,
	   class _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
    class unordered_map

    template<class _Value,
	   class _Hash = hash<_Value>,
	   class _Pred = std::equal_to<_Value>,
	   class _Alloc = std::allocator<_Value> >
    class unordered_multiset


    template<class _Key, class _Tp,
	   class _Hash = hash<_Key>,
	   class _Pred = std::equal_to<_Key>,
	   class _Alloc = std::allocator<std::pair<const _Key, _Tp> > >
    class unordered_multimap
  
```
其实现都是对hashtable的封装
![](../../../images/c%26c%2B%2B/stl102.png)
![](../../../images/c%26c%2B%2B/stl103.png)


![](../../../images/c%26c%2B%2B/stl33.png)

迭代器的分类
![](../../../images/c%26c%2B%2B/stl34.png)
这些类型都是空class,如果创建这些class的对象，对象需要占用一个字节的。如果这些class被继承，派生类的对象中，空基类的大小是0
![](../../../images/c%26c%2B%2B/stl35.png)


![](../../../images/c%26c%2B%2B/stl36.png)
各种不同版本的istream_iterator的实现，其iterator_category都是input_iterator_tag

![](../../../images/c%26c%2B%2B/stl37.png)
各种不同版本ostream_iterator的实现，其iterator_category都是output_iterator_tag

![](../../../images/c%26c%2B%2B/stl38.png)

![](../../../images/c%26c%2B%2B/stl39.png)

最后，stl中的copy算法，其G2.8版本实现如下:
```c++
template <class InputIterator, class OutputIterator>
inline OutputIterator copy(InputIterator first, InputIterator last,
                           OutputIterator result)
{
  return __copy_dispatch<InputIterator,OutputIterator>()(first, last, result);
}
```
以及针对`char*`和 `wchar_t*`类型的重载
```c++
inline char* copy(const char* first, const char* last, char* result) {
  memmove(result, first, last - first);
  return result + (last - first);
}

inline wchar_t* copy(const wchar_t* first, const wchar_t* last,
                     wchar_t* result) {
  memmove(result, first, sizeof(wchar_t) * (last - first));
  return result + (last - first);
}
```

__copy_dispatch实现如下,有类模板和针对模板参数的偏特化

```c++
template <class InputIterator, class OutputIterator>
struct __copy_dispatch
{
  OutputIterator operator()(InputIterator first, InputIterator last,
                            OutputIterator result) {
    return __copy(first, last, result, iterator_category(first));
  }
};

template <class T>
struct __copy_dispatch<T*, T*>
{
  T* operator()(T* first, T* last, T* result) {
    typedef typename __type_traits<T>::has_trivial_assignment_operator t; 
    return __copy_t(first, last, result, t());
  }
};

template <class T>
struct __copy_dispatch<const T*, T*>
{
  T* operator()(const T* first, const T* last, T* result) {
    typedef typename __type_traits<T>::has_trivial_assignment_operator t; 
    return __copy_t(first, last, result, t());
  }
};
```

__copy的实现，有两种重载，分别对应于iterator_category是random_access_iterator_tag和input_iterator_tag
```c++
template <class RandomAccessIterator, class OutputIterator>
inline OutputIterator 
__copy(RandomAccessIterator first, RandomAccessIterator last,
       OutputIterator result, random_access_iterator_tag)
{
  return __copy_d(first, last, result, distance_type(first));
}

template <class InputIterator, class OutputIterator>
inline OutputIterator __copy(InputIterator first, InputIterator last,
                             OutputIterator result, input_iterator_tag)
{
  for ( ; first != last; ++result, ++first)
    *result = *first;
  return result;
}
```
__copy_d的实现如下
```c++
template <class RandomAccessIterator, class OutputIterator, class Distance>
inline OutputIterator
__copy_d(RandomAccessIterator first, RandomAccessIterator last,
         OutputIterator result, Distance*)
{
  for (Distance n = last - first; n > 0; --n, ++result, ++first) 
    *result = *first;
  return result;
}
```

而__copy_t的实现，针对__type_traits<T>::has_trivial_assignment_operator的两种结果，有如下两个重载
```c++

template <class T>
inline T* __copy_t(const T* first, const T* last, T* result, __true_type) {
  memmove(result, first, sizeof(T) * (last - first));
  return result + (last - first);
}

template <class T>
inline T* __copy_t(const T* first, const T* last, T* result, __false_type) {
  return __copy_d(first, last, result, (ptrdiff_t*) 0);
}

```

总结如下图：
![](../../../images/c%26c%2B%2B/stl40.png)



destroy算法G2.8实现如下

```c++

  template <typename _ForwardIterator>
    inline void
    destroy(_ForwardIterator __first, _ForwardIterator __last)
    {
      std::_Destroy(__first, __last);
    }

  template<typename _ForwardIterator>
    inline void
    _Destroy(_ForwardIterator __first, _ForwardIterator __last)
    {
      typedef typename iterator_traits<_ForwardIterator>::value_type
                       _Value_type;

      std::_Destroy_aux<__has_trivial_destructor(_Value_type)>::
	__destroy(__first, __last);
    }


    template<bool>
    struct _Destroy_aux
    {
      template<typename _ForwardIterator>
	      static  void
	      __destroy(_ForwardIterator __first, _ForwardIterator __last)
	      {
	        for (; __first != __last; ++__first)
	        std::_Destroy(std::__addressof(*__first));
	      }
    };

  template<>
    struct _Destroy_aux<true>
    {
      template<typename _ForwardIterator>
        static void
        __destroy(_ForwardIterator, _ForwardIterator) { }
    };


    template<typename _Tp>
    inline void
    _Destroy(_Tp* __pointer)
    {
      __pointer->~_Tp();
    }

```

unique_copy的G2.8实现如下
```c++
template <class InputIterator, class OutputIterator>
inline OutputIterator unique_copy(InputIterator first, InputIterator last,
                                  OutputIterator result) {
  if (first == last) return result;
  return __unique_copy(first, last, result, iterator_category(result));
}


template <class InputIterator, class ForwardIterator>
ForwardIterator __unique_copy(InputIterator first, InputIterator last,
                              ForwardIterator result, forward_iterator_tag) {
  *result = *first;
  while (++first != last)
    if (*result != *first) *++result = *first;
  return ++result;
}


template <class InputIterator, class OutputIterator>
inline OutputIterator __unique_copy(InputIterator first, InputIterator last,
                                    OutputIterator result, 
                                    output_iterator_tag) {
  return __unique_copy(first, last, result, value_type(first));
}


template <class InputIterator, class OutputIterator, class T>
OutputIterator __unique_copy(InputIterator first, InputIterator last,
                             OutputIterator result, T*) {
  T value = *first;
  *result = value;
  while (++first != last)
    if (value != *first) {
      value = *first;
      *++result = value;
    }
  return ++result;
}



```

![](../../../images/c%26c%2B%2B/stl42.png)


下面看几个算法
![](../../../images/c%26c%2B%2B/stl43.png)
![](../../../images/c%26c%2B%2B/stl44.png)
![](../../../images/c%26c%2B%2B/stl45.png)
还有个replace_copy_if
```c++
template <class Iterator, class OutputIterator, class Predicate, class T>
OutputIterator replace_copy_if(Iterator first, Iterator last,
                               OutputIterator result, Predicate pred,
                               const T& new_value) {
  for ( ; first != last; ++first, ++result)
    *result = pred(*first) ? new_value : *first;
  return result;
}
```
![](../../../images/c%26c%2B%2B/stl46.png)
![](../../../images/c%26c%2B%2B/stl47.png)
![](../../../images/c%26c%2B%2B/stl48.png)
<font color=red>unordered_map, unordered_set 等哈希容器遍历自然形成sorted状态？？怎么实现的？？</font>
![](../../../images/c%26c%2B%2B/stl49.png)
![](../../../images/c%26c%2B%2B/stl50.png)



仿函数：
stl中提供了3大类仿函数：算数类仿函数，逻辑运算类仿函数，关系类仿函数，如下
![](../../../images/c%26c%2B%2B/stl51.png)
所谓的仿函数，就是可调用对象。
```c++
//算数类
template <class T>
struct plus : public binary_function<T, T, T> {
    T operator()(const T& x, const T& y) const { return x + y; }
};

template <class T>
struct minus : public binary_function<T, T, T> {
    T operator()(const T& x, const T& y) const { return x - y; }
};

template <class T>
struct multiplies : public binary_function<T, T, T> {
    T operator()(const T& x, const T& y) const { return x * y; }
};

template <class T>
struct divides : public binary_function<T, T, T> {
    T operator()(const T& x, const T& y) const { return x / y; }
};



template <class T>
struct modulus : public binary_function<T, T, T> {
    T operator()(const T& x, const T& y) const { return x % y; }
};

template <class T>
struct negate : public unary_function<T, T> {
    T operator()(const T& x) const { return -x; }
};

//关系类
template <class T>
struct equal_to : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x == y; }
};

template <class T>
struct not_equal_to : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x != y; }
};

template <class T>
struct greater : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x > y; }
};

template <class T>
struct less : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x < y; }
};

template <class T>
struct greater_equal : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x >= y; }
};

template <class T>
struct less_equal : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x <= y; }
};

//逻辑运算类
template <class T>
struct logical_and : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x && y; }
};

template <class T>
struct logical_or : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x || y; }
};

template <class T>
struct logical_not : public unary_function<T, bool> {
    bool operator()(const T& x) const { return !x; }
};

```
gnu c++ 提供了如下几个非标准库规定的仿函数
![](../../../images/c%26c%2B%2B/stl52.png)



那如果我们自己写一个仿函数
```c++
struct myclass{
  bool operator(int i,int j){
    return i<j;
  }
} mlc;
```
这时候mlc也是可调用的，都可以被算法作为断言使用，如下所示
![](../../../images/c%26c%2B%2B/stl53.png)


但是我们创建的这个mlc并没有融入stl中，因为它没有继承 unary_function或者binary_function.
```c++
template <class Arg, class Result>
struct unary_function {
    typedef Arg argument_type;
    typedef Result result_type;
};

template <class Arg1, class Arg2, class Result>
struct binary_function {
    typedef Arg1 first_argument_type;
    typedef Arg2 second_argument_type;
    typedef Result result_type;
};      
```
只有当仿函数继承了unary_function或者binary_function时，它才可被适配器适配
![](../../../images/c%26c%2B%2B/stl54.png)

适配器：
stl中有3中适配器：容器适配器，迭代器适配器，仿函数适配器。
适配器的实现，本身就是个类，被适配的对象，比如本来的容器，迭代器，仿函数作为类成员，适配器在该成员之上再封装新的接口。

容器适配器之前已经看过了
![](../../../images/c%26c%2B%2B/stl55.png)

仿函数适配器：
binder2nd
![](../../../images/c%26c%2B%2B/stl56.png)

unary_negate:
![](../../../images/c%26c%2B%2B/stl57.png)


binder1st:
```c++
template <class Operation> 
class binder1st
  : public unary_function<typename Operation::second_argument_type,
                          typename Operation::result_type> {
protected:
  Operation op;
  typename Operation::first_argument_type value;
public:
  binder1st(const Operation& x,
            const typename Operation::first_argument_type& y)
      : op(x), value(y) {}
      
  typename Operation::result_type
  operator()(const typename Operation::second_argument_type& x) const {
    return op(value, x); 
  }
};

template <class Operation, class T>
inline binder1st<Operation> bind1st(const Operation& op, const T& x) {
  typedef typename Operation::first_argument_type arg1_type;
  return binder1st<Operation>(op, arg1_type(x));
}

```

c++11提供的函数适配器 bind

![](../../../images/c%26c%2B%2B/stl58.png)

迭代器适配器 
![](../../../images/c%26c%2B%2B/stl59.png)
![](../../../images/c%26c%2B%2B/stl60.png)
G2.8代码
```c++
template <class Container>
inline front_insert_iterator<Container> front_inserter(Container& x) {
  return front_insert_iterator<Container>(x);
}

template <class Container>
class insert_iterator {
protected:
  Container* container;
  typename Container::iterator iter;
public:
  typedef output_iterator_tag iterator_category;
  typedef void                value_type;
  typedef void                difference_type;
  typedef void                pointer;
  typedef void                reference;

  insert_iterator(Container& x, typename Container::iterator i) 
    : container(&x), iter(i) {}
  insert_iterator<Container>&
  operator=(const typename Container::value_type& value) { 
    iter = container->insert(iter, value);
    ++iter;
    return *this;
  }
  insert_iterator<Container>& operator*() { return *this; }
  insert_iterator<Container>& operator++() { return *this; }
  insert_iterator<Container>& operator++(int) { return *this; }
};
```

ostream_iterator，G2.8,这个版本算是古早版本了
```c++
template <class T>
class ostream_iterator {
protected:
  ostream* stream;
  const char* string;
public:
  typedef output_iterator_tag iterator_category;
  typedef void                value_type;
  typedef void                difference_type;
  typedef void                pointer;
  typedef void                reference;

  ostream_iterator(ostream& s) : stream(&s), string(0) {}
  ostream_iterator(ostream& s, const char* c) : stream(&s), string(c)  {}
  ostream_iterator<T>& operator=(const T& value) { 
    *stream << value;
    if (string) *stream << string;
    return *this;
  }
  ostream_iterator<T>& operator*() { return *this; }
  ostream_iterator<T>& operator++() { return *this; } 
  ostream_iterator<T>& operator++(int) { return *this; } 
};
```
![](../../../images/c%26c%2B%2B/stl61.png)

istream_iteraor，G2.8,这个版本算是古早版本了
```c++
template <class T, class Distance = ptrdiff_t> 
class istream_iterator {
  friend bool
  operator==(const istream_iterator<T, Distance>& x,
                                   const istream_iterator<T, Distance>& y);
protected:
  istream* stream;
  T value;
  bool end_marker; //end_marker 为true,表示有数据要读，end_marker为false,表示没有数据要读了
  void read() {
    end_marker = (*stream) ? true : false; //cin 应该有operator bool转换，但我没有找到，只找到 operator int,实现的也不是很合理
    if (end_marker) *stream >> value;
    end_marker = (*stream) ? true : false;
  }
public:
  typedef input_iterator_tag iterator_category;
  typedef T                  value_type;
  typedef Distance           difference_type;
  typedef const T*           pointer;
  typedef const T&           reference;

  istream_iterator() : stream(&cin), end_marker(false) {}
  istream_iterator(istream& s) : stream(&s) { read(); }
  reference operator*() const { return value; }

  pointer operator->() const { return &(operator*()); }

  istream_iterator<T, Distance>& operator++() { 
    read(); 
    return *this;
  }
  istream_iterator<T, Distance> operator++(int)  {
    istream_iterator<T, Distance> tmp = *this;
    read();
    return tmp;
  }
};

template <class T, class Distance>
inline bool operator==(const istream_iterator<T, Distance>& x,
                       const istream_iterator<T, Distance>& y) {
  return x.stream == y.stream && x.end_marker == y.end_marker ||
         x.end_marker == false && y.end_marker == false;
}

template <class T, class Distance>
inline input_iterator_tag 
iterator_category(const istream_iterator<T, Distance>&) {
  return input_iterator_tag();
}

template <class T, class Distance>
inline T* value_type(const istream_iterator<T, Distance>&) { return (T*) 0; }

template <class T, class Distance>
inline Distance* distance_type(const istream_iterator<T, Distance>&) {
  return (Distance*) 0;
}


```

![](../../../images/c%26c%2B%2B/stl62.png)
![](../../../images/c%26c%2B%2B/stl63.png)

对于对象类型，当我们需要求其hash值时，一个简单的做法如下
```c++
struct Customer{
std::string fname;
std::string lname;
int age;
};

class CustomerHash{
  std::size_t operator()(const Customer& c)const{
    return std::hash<std::string>()(c.fname) + 
          std::hash<std::string>()(c.lname)+
          std::hash<int>()(c.age);
  }
};
```

这样简单的相加，并不能保证hash的结果非常随机。

一个更好的万用hash函数如下
![](../../../images/c%26c%2B%2B/stl104.png)
![](../../../images/c%26c%2B%2B/stl105.png)
![](../../../images/c%26c%2B%2B/stl107.png)


tuple
![](../../../images/c%26c%2B%2B/stl106.png)
![](../../../images/c%26c%2B%2B/stl108.png)

type traits
G2.8
```c++
struct __true_type {
};

struct __false_type {
};

template <class type>
struct __type_traits { 
   typedef __true_type     this_dummy_member_must_be_first;
   typedef __false_type    has_trivial_default_constructor;
   typedef __false_type    has_trivial_copy_constructor;
   typedef __false_type    has_trivial_assignment_operator;
   typedef __false_type    has_trivial_destructor;
   typedef __false_type    is_POD_type;
};

```
并提供了内置类型的type tratis特化
```c++
__STL_TEMPLATE_NULL struct __type_traits<char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
__STL_TEMPLATE_NULL struct __type_traits<int> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
__STL_TEMPLATE_NULL struct __type_traits<float> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<double> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
//......
```

以及指针类型的偏特化
```c++
template <class T>
struct __type_traits<T*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
```

但是如果不是内置类型，其type traits中的各个associate type就要自己提供了。

c++11提供了更为通用的type traits,对于非内置类型不需要自己提供实现
![](../../../images/c%26c%2B%2B/stl109.png)
![](../../../images/c%26c%2B%2B/stl110.png)

c++11 type_traits的使用如下
![](../../../images/c%26c%2B%2B/stl111.png)
![](../../../images/c%26c%2B%2B/stl112.png)
![](../../../images/c%26c%2B%2B/stl113.png)
![](../../../images/c%26c%2B%2B/stl114.png)
![](../../../images/c%26c%2B%2B/stl115.png)

is_void实现
```c++

  template<typename _Tp>
    struct is_void
    : public __is_void_helper<typename remove_cv<_Tp>::type>::type
    { };


  template<typename>
    struct __is_void_helper
    : public false_type { };

  template<>
    struct __is_void_helper<void>
    : public true_type { };
    
    /// The type used as a compile-time boolean with true value.
  typedef integral_constant<bool, true>     true_type;

  /// The type used as a compile-time boolean with false value.
  typedef integral_constant<bool, false>    false_type;


  template<typename _Tp, _Tp __v>
    struct integral_constant
    {
      static constexpr _Tp                  value = __v;
      typedef _Tp                           value_type;
      typedef integral_constant<_Tp, __v>   type;
      constexpr operator value_type() const { return value; }
      constexpr value_type operator()() const { return value; }
    };

```
其中remove_cv实现如下
```c++
  /// remove_cv
  template<typename _Tp>
    struct remove_cv
    {
      typedef typename
      remove_const<typename remove_volatile<_Tp>::type>::type     type;
    };


  /// remove_volatile
  template<typename _Tp>
    struct remove_volatile
    { typedef _Tp     type; };

  template<typename _Tp>
    struct remove_volatile<_Tp volatile>
    { typedef _Tp     type; };


  /// remove_const
  template<typename _Tp>
    struct remove_const
    { typedef _Tp     type; };

  template<typename _Tp>
    struct remove_const<_Tp const>
    { typedef _Tp     type; };
  
```

is_integral的实现
![](../../../images/c%26c%2B%2B/stl118.png)

以上这些实现都是完全在标准库中完成的。

有部分实现，标准库中没有代码，应该是在编译器层面实现的
![](../../../images/c%26c%2B%2B/stl119.png)