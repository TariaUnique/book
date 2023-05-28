Random access iterators support the following operations:



Increment/Decrement (++, --): Move the iterator to the next/previous element in the container.

Equality/Inequality (==, !=): Compare two iterators for equality/inequality.

Dereference (*): Access the value pointed to by the iterator.

Addition/Subtraction of integer values (+, -): Move the iterator forward or backward a specified number of positions in the container.

Compound addition/subtraction assignment (+=, -=): Move the iterator forward or backward a specified number of positions in the container.

Difference (it1 - it2): Calculate the distance between two iterators.

Comparison (<, >, <=, >=): Compare two iterators' positions in the container relative to each other.

Access elements at an arbitrary offset via an index operator ([]): Access elements a specific number of positions away (offset) from the iterator.


These operations make random access iterators the most powerful among iterator categories in C++. They can be used in various STL algorithms (e.g., std::sort) and containers that provide random access (e.g., std::vector, std::deque).

```c++
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};

    // Increment/Decrement
    std::vector<int>::iterator it1 = nums.begin();
    ++it1;
    --it1;

    // Equality/Inequality
    bool equal = it1 == nums.begin();
    bool not_equal = it1 != nums.end();

    // Dereference
    int value = *it1;

    // Addition/Subtraction
    it1 = it1 + 2;
    it1 = it1 - 1;

    // Compound addition/subtraction assignment
    it1 += 1;
    it1 -= 1;

    // Difference
    std::vector<int>::iterator it2 = nums.begin() + 4;
    int difference = it2 - it1;

    // Comparison
    bool less = it1 < it2;
    bool greater = it1 > it2;
    bool less_equal = it1 <= it2;
    bool greater_equal = it1 >= it2;

    // Access elements at an arbitrary offset via an index operator
    int offset_value = it1[2];

    // Sort using random access iterators
    std::sort(nums.begin(), nums.end());

    return 0;
}
```