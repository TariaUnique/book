```c++
#include <iostream>
#include <memory>
#include <initializer_list>
#include <stdexcept>

template <typename T, typename Allocator = std::allocator<T>>
class CustomVector {
public:
    using iterator = T*;
    using const_iterator = const T*;
    using value_type = T;
    using size_type = size_t;

    CustomVector(size_t initial_size = 0) : data_allocator(Allocator()) {
        resize(initial_size);
    }

    CustomVector(const std::initializer_list<T>& init_list)
        : data_allocator(Allocator()) {
        reserve(init_list.size());
        for (const T& value : init_list) {
            push_back(value);
        }
    }

    CustomVector(const CustomVector& other)
        : data_allocator(Allocator()) {
        reserve(other.size_);
        for (size_t i = 0; i < other.size_; ++i) {
            push_back(other.data[i]);
        }
    }

    CustomVector(CustomVector&& other)
        : data(other.data), size_(other.size_), capacity_(other.capacity_), data_allocator(std::move(other.data_allocator)) {
        other.data = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }

    ~CustomVector() {
        clear();
        data_allocator.deallocate(data, capacity_);
    }

    CustomVector& operator=(const CustomVector& other) {
        if (&other == this) { return *this; }

        clear();
        data_allocator.deallocate(data, capacity_);
        reserve(other.size_);
        for (size_t i = 0; i < other.size_; ++i) {
            push_back(other.data[i]);
        }
        return *this;
    }

    CustomVector& operator=(CustomVector&& other) {
        if (&other == this) { return *this; }

        clear();
        data_allocator.deallocate(data, capacity_);

        data = other.data;
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_allocator = std::move(other.data_allocator);

        other.data = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;

        return *this;
    }

    void push_back(const T& value) {
        if (size_ == capacity_) {
            reserve(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        data_allocator.construct(data + size_, value);
        ++size_;
    }

    void pop_back() {
        if (size_ == 0) {
            throw std::out_of_range("pop_back on empty CustomVector");
        }
        data_allocator.destroy(data + size_ - 1);
        --size_;
    }

    T& operator[](size_t index) { return data[index]; }
    const T& operator[](size_t index) const { return data[index]; }

    T& at(size_t index) {
        if (index >= size_) {
            throw std::out_of_range("CustomVector index out of range");
        }
        return data[index];
    }

    const T& at(size_t index) const {
        if (index >= size_) {
            throw std::out_of_range("CustomVector index out of range");
        }
        return data[index];
    }

    size_t size() const { return size_; }
    size_t capacity() const { return capacity_; }

    bool empty() const { return size_ == 0; }

    iterator begin() { return data; }
    const_iterator begin() const { return data; }
    const_iterator cbegin() const { return data; }

    iterator end() { return data + size_; }
    const_iterator end() const { return data + size_; }
    const_iterator cend() const { return data + size_; }

    void clear() {
        while (size_ > 0) {
            pop_back();
        }
    }

    void resize(size_t new_size) {
        if (new_size > capacity_) {
            reserve(new_size);
        }
        if (new_size < size_) {
            for (size_t i = new_size; i < size_; ++i) {
                data_allocator.destroy(data + i);
            }
        } else if (new_size > size_) {
            for (size_t i = size_; i < new_size; ++i) {
                data_allocator.construct(data + i);
            }
        }
        size_ = new_size;
    }

    void reserve(size_t new_capacity) {
        if (new_capacity > capacity_) {
            T* new_data = data_allocator.allocate(new_capacity);
            for (size_t i = 0; i < size_; ++i) {
                data_allocator.construct(new_data + i, std::move_if_noexcept(data[i]));
                data_allocator.destroy(data + i);
            }
            data_allocator.deallocate(data, capacity_);
            data = new_data;
            capacity_ = new_capacity;
        }
    }

    void shrink_to_fit() {
        if (size_ < capacity_) {
            if (data) {
                data_allocator.deallocate(data + size_, capacity_ - size_);
            }
            capacity_ = size_;
        }
    }

private:
    T* data = nullptr;
    size_t size_ = 0;
    size_t capacity_ = 0;
    Allocator data_allocator;
};

void test_custom_vector() {
    CustomVector<int> vec1{1, 2, 3};
    for (int value : vec1) {
        std::cout << value << " ";
    }
    std::cout << std::endl; // Output: 1 2 3

    CustomVector<int> vec2(std::move(vec1));
    for (int value : vec2) {
        std::cout << value << \" ";
    }
    std::cout << std::endl; // Output: 1 2 3
}

int main() {
    test_custom_vector();
    return 0;
}
```