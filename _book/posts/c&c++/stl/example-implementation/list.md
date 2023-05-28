```c++
#include <iterator>
#include <memory>
#include <initializer_list>

template<typename T, typename Allocator = std::allocator<T>>
class DoublyLinkedList {
private:
    struct Node {
        T value;
        Node* prev;
        Node* next;

        Node()
            : value(T()), prev(nullptr), next(nullptr) {}

        Node(const T& val)
            : value(val), prev(nullptr), next(nullptr) {}

        Node(T&& val)
            : value(std::move(val)), prev(nullptr), next(nullptr) {}
    };
    //std::allocator_traits<Allocator>::template 表示rebind_alloc<>是std::allocator_traits<Allocator>中的templete member, 这个templete member是个type name 
    using NodeAlloc = typename std::allocator_traits<Allocator>::template rebind_alloc<Node>;
    NodeAlloc alloc;
    Node* head;
    Node* tail;
    std::size_t size;

    Node* create_node(const T& value) {
        Node* new_node = alloc.allocate(1);
        alloc.construct(new_node, value);
        return new_node;
    }

    void destroy_node(Node* node) {
        alloc.destroy(node);
        alloc.deallocate(node, 1);
    }

public:
    DoublyLinkedList()
        : head(nullptr), tail(nullptr), size(0) {}

    ~DoublyLinkedList() {
        clear();
    }

    // Initialization with an initializer list.
    DoublyLinkedList(std::initializer_list<T> init) : DoublyLinkedList() {
        for (const T& value : init) { push_back(value); }
    }

    // Copy constructor
    DoublyLinkedList(const DoublyLinkedList& other) : DoublyLinkedList() {
        Node* cur_node = other.head;
        for (std::size_t i = 0; i < other.size; ++i) {
            push_back(cur_node->value);
            cur_node = cur_node->next;
        }
    }

    // Move constructor
    DoublyLinkedList(DoublyLinkedList&& other) noexcept
        : head(std::move(other.head)),
          tail(std::move(other.tail)),
          size(std::move(other.size)) {
        other.head = nullptr;
        other.tail = nullptr;
        other.size = 0;
    }

    // Assignment operators
    DoublyLinkedList& operator=(const DoublyLinkedList& other) {
        if (this != &other) {
            clear();
            Node* cur_node = other.head;
            for (size_t i = 0; i < other.size; ++i) {
                push_back(cur_node->value);
                cur_node = cur_node->next;
            }
        }
        return *this;
    }

    DoublyLinkedList& operator=(DoublyLinkedList&& other) noexcept {
        if (this != &other) {
            clear();
            head = std::move(other.head);
            tail = std::move(other.tail);
            size = std::move(other.size);
            other.head = nullptr;
            other.tail = nullptr;
            other.size = 0;
        }
        return *this;
    }

    // Adds value to the end of the list
    void push_back(const T& value) {
        Node* new_node = create_node(value);
        if (tail) {
            tail->next = new_node;
            new_node->prev = tail;
            tail = new_node;
        } else {
            head = tail = new_node;
        }
        size++;
    }

    // Removes last element and returns its value
    T pop_back() {
        Node* old_tail = tail;
        tail = tail->prev;
        if (tail) { tail->next = nullptr; }
        else { head = nullptr; }
        T value = std::move(old_tail->value);
        destroy_node(old_tail);
        size--;
        return value;
    }

    // Adds value to the beginning of the list
    void push_front(const T& value) {
        Node* new_node = create_node(value);
        if (head) {
            head->prev = new_node;
            new_node->next = head;
            head = new_node;
        } else {
            head = tail = new_node;
        }
        size++;
    }

    // Removes first element and returns its value
    T pop_front() {
        Node* old_head = head;
        head = head->next;
        if (head) { head->prev = nullptr; }
        else { tail = nullptr; }
        T value = std::move(old_head->value);
        destroy_node(old_head);
        size--;
        return value;
    }

    // Removes all elements from the list
    void clear() {
        while (size != 0) { pop_front(); }
    }

    // Returns the number of elements in the list
    std::size_t length() const {
        return size;
    }

    // Iterator class
    class iterator {
    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;

    private:
        Node* current_node;

    public:
        iterator(Node* node) : current_node(node) {}

        reference operator*() const { return current_node->value; }
        pointer operator->() const { return &current_node->value; }

        iterator& operator++() {
            current_node = current_node->next;
            return *this;
        }

        iterator operator++(int) {
            iterator temp = *this;
            current_node = current_node->next;
            return temp;
        }

        iterator& operator--() {
            current_node = current_node->prev;
            return *this;
        }

        iterator operator--(int) {
            iterator temp = *this;
            current_node = current_node->prev;
            return temp;
        }

        bool operator==(const iterator& other) const {
            return current_node == other.current_node;
        }

        bool operator!=(const iterator& other) const {
            return current_node != other.current_node;
        }
    };

    iterator begin() { return iterator(head); }
    iterator end() { return iterator(nullptr); }
};
```