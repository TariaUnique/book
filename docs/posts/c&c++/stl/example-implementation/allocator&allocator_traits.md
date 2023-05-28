simple allcator use `::operator new` and `::operator delete`
```c++
#include <memory>

template <typename T>
class MyAllocator {
public:
    using value_type = T;
    using pointer = T*;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;

    template <typename U>
    struct rebind {
        using other = MyAllocator<U>;
    };

    MyAllocator() = default;

    pointer allocate(size_type n) {
        return static_cast<pointer>(::operator new(n * sizeof(value_type)));
    }

    void deallocate(pointer p, size_type) noexcept {
        ::operator delete(p);
    }

    template <typename U, typename... Args>
    void construct(U* p, Args&&... args) {
        new (p) U(std::forward<Args>(args)...);
    }

    template<typename U>
    void destroy(U* p) {
        p->~U();
    }

    size_type max_size() const noexcept {
        return std::numeric_limits<size_type>::max() / sizeof(value_type);
    }
};
```
This custom allocator uses the `operator new` and `operator delete` to allocate and deallocate memory, respectively. It provides the nested `rebind<U>::other` template, which is necessary for the `rebind_alloc` alias template in `allocator_traits` to work correctly

simplified implementation of allocator_traits
```c++
template<typename Allocator>
struct allocator_traits {
    using allocator_type                = Allocator;
    using value_type                    = typename Allocator::value_type;
    using pointer                       = typename Allocator::pointer;
    using size_type                     = typename Allocator::size_type;
    using difference_type               = typename Allocator::_difference_type;

    //Allocator::template 表示rebind<>是Allocator中的templete member。 templete member就是在member之上加了模板参数，这个member可以是sub class, 可以是type name,可以是data member.
    
    //typename 表示 rebind<U>中的other是个type,not a member
    template<typename U>
    using rebind_alloc = typename Allocator::template rebind<U>::other; 

    template<typename U>
    using rebind_traits = allocator_traits<rebind_alloc<U>>;

    static pointer allocate(Allocator& a, size_type n) {
        return a.allocate(n);
    }
    
    template<typename U, typename... Args>
    static void construct(Allocator& a, U* p, Args&&... args) {
        a.construct(p, std::forward<Args>(args)...);
    }

    static void deallocate(Allocator& a, pointer p, size_type n) {
        a.deallocate(p, n);
    }

    template<typename U>
    static void destroy(Allocator& a, U* p) {
        a.destroy(p);
    }

    static size_type max_size(const Allocator& a) noexcept {
        return a.max_size();
    }
};
```

The purpose of allocator_traits is to provide a standardized interface for working with various allocator types. It includes a set of static functions, constants, and type aliases that makes it easier for containers and other components to use allocators in a generic way.


By using allocator_traits, components can work with both the standard allocator and custom allocators as long as they meet the requirements. This simplifies the code and reduces the reliance on specific allocator types, ultimately resulting in more flexible and maintainable code.


Additionally, allocator_traits helps to fill in any missing functionality in the allocator by providing default implementations for certain functions or types. It ensures that the allocator has a complete and uniform interface, regardless of its actual implementation.

解释下以上模板类中的typename 关键字的作用：

假设代码如下
```c++
template <typename T, typename Allocator>
class MyContainer {
public:
    using value_type = T;
    using pointer = typename Allocator::pointer;
    // ...

private:
    Allocator allocator;
};
```
其中`using pointer = typename Allocator::pointer;`这个语句中的typename，表示其后的pointer是个type.

在这个类中Allocator是个模板参数，在编译器编译到这里时，并不知道Allocator的具体类型，那么如果只有`Allocator::pointer`,这个pointer也可能是个static member, 这里加上typename,表示这个pointer是个type。

而在`using value_type = T;`中，不需要使用typename关键字，因为编译器已经知道，T就是一个type.


