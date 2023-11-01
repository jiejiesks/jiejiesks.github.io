# STL


# STL容器

## STL序列式容器

### array数组

array 容器是 **[C++ 11](https://haicoder.net/cpp/cpp-tutorial.html)** 标准中新增的 **[序列容器](https://haicoder.net/stl/stl-sequential-container.html)**，简单地理解，它就是在 C++ 普通数组的基础上，添加了一些成员函数和全局函数。在使用上，它比普通数组更安全，且效率并没有因此变差。

和其它容器不同，array 容器的大小是固定的，无法动态的==扩展==或==收缩==，这也就意味着，在使用该容器的过程无法借由增加或移除元素而改变其大小，它只允许访问或者替换存储的元素。

在 `array<T,N>` 类模板中，T 用于指明容器中的存储的具体数据类型，N 用于指明容器的大小，需要注意的是，这里的 N 必须是常量，不能用变量表示。

```
std::array<double, 10> values;
```

array的初始化赋值语法

```
std::array<double, 10> values {0.1,1.4,1.8,,2.4};
```

这里只初始化了前 4 个元素，剩余的元素都会被初始化为 0.0。

| 成员函数            | 功能                                                         |
| ------------------- | ------------------------------------------------------------ |
| begin()             | 返回指向容器中第一个元素的随机访问迭代器。                   |
| end()               | 返回指向容器最后一个元素之后一个位置的随机访问迭代器，通常和 begin() 结合使用。 |
| rbegin()            | 返回指向最后一个元素的随机访问迭代器。                       |
| rend()              | 返回指向第一个元素之前一个位置的随机访问迭代器。             |
| cbegin()            | 和 begin() 功能相同，只不过在其基础上增加了 const 属性，不能用于修改元素。 |
| cend()              | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crbegin()           | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crend()             | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| size()              | 返回容器中当前元素的数量，其值始终等于初始化 array 类的第二个模板参数 N。 |
| max_size()          | 返回容器可容纳元素的最大数量，其值始终等于初始化 array 类的第二个模板参数 N。 |
| empty()             | 判断容器是否为空，和通过 size()==0 的判断条件功能相同，但其效率可能更快。 |
| at(n)               | 返回容器中 n 位置处元素的引用，该函数自动检查 n 是否在有效的范围内，如果不是则抛出 out_of_range 异常。 |
| front()             | 返回容器中第一个元素的直接引用，该函数不适用于空的 array 容器。 |
| back()              | 返回容器中最后一个元素的直接应用，该函数同样不适用于空的 array 容器。 |
| data()              | 返回一个指向容器首个元素的指针。利用该指针，可实现复制容器中所有元素等类似功能。 |
| fill(val)           | 将 val 这个值赋值给容器中的每个元素。                        |
| array1.swap(array2) | 交换 array1 和 array2 容器中的所有元素，但前提是它们具有相同的长度和类型。 |

### vector容器

vector 容器是 **[STL](https://haicoder.net/stl/stl-tutorial.html)** 中最常用的容器之一，它和 array 容器非常类似，都可以看做是对 **[C++](https://haicoder.net/cpp/cpp-tutorial.html)** 普通数组的 “升级版”。不同之处在于，array 实现的是==静态数组==（容量固定的数组），而 vector 实现的是一个==动态数组==，即可以进行元素的插入和删除，在此过程中，vector 会动态调整所占用的内存空间，整个过程无需人工干预。

vector 常被称为向量容器，因为该容器擅长在尾部插入或删除元素，在==常量时间内就可以完成==，时间复杂度为 O(1)；而对于在容器头部或者中部插入或删除元素，则花费时间要长一些（移动元素需要耗费时间），时间复杂度为线性阶 O(n)。

vector初始化

```
std::vector<int> numbers = {1, 2, 3, 4, 5};		
```



| 函数成员         | 函数功能                                                     |
| ---------------- | ------------------------------------------------------------ |
| begin()          | 返回指向容器中第一个元素的迭代器。                           |
| end()            | 返回指向容器最后一个元素所在位置后一个位置的迭代器，通常和 begin() 结合使用。 |
| rbegin()         | 返回指向最后一个元素的迭代器。                               |
| rend()           | 返回指向第一个元素所在位置前一个位置的迭代器。               |
| cbegin()         | 和 begin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| cend()           | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crbegin()        | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crend()          | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| ==size()==       | 返回实际元素个数。                                           |
| max_size()       | 返回元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 |
| resize()         | 改变实际元素的个数。                                         |
| capacity()       | 返回当前容量。                                               |
| ==empty()==      | 判断容器中是否有元素，若无元素，则返回 true；反之，返回 false。 |
| reserve()        | 增加容器的容量。                                             |
| shrink _to_fit() | 将内存减少到等于当前元素实际所使用的大小。                   |
| operator[ ]      | 重载了 [ ] 运算符，可以向访问数组中元素那样，通过下标即可访问甚至修改 vector 容器中的元素。 |
| at()             | 使用经过边界检查的索引访问元素。                             |
| front()          | 返回第一个元素的引用。                                       |
| back()           | 返回最后一个元素的引用。                                     |
| data()           | 返回指向容器中第一个元素的指针。                             |
| assign()         | 用新元素替换原有内容。                                       |
| push_back()      | 在序列的尾部添加一个元素。                                   |
| pop_back()       | 移出序列尾部的元素。                                         |
| insert()         | 在指定的位置插入一个或多个元素。                             |
| erase()          | 移出一个元素或一段元素。                                     |
| clear()          | 移出所有的元素，容器大小变为 0。                             |
| swap()           | 交换两个容器的所有元素。                                     |
| emplace()        | 在指定的位置直接生成一个元素。                               |
| emplace_back()   | 在序列尾部生成一个元素。                                     |

### deque队列

**[STL](https://haicoder.net/stl/stl-tutorial.html)** 中的 deque 是双端队列容器，deque 容器和 **[vector](https://haicoder.net/stl/stl-vector.html)** 容器有很多相似之处，deque 容器也==擅长在序列尾部添加或删除元素==（时间复杂度为O(1)），而不擅长在序列中间添加或删除元素。deque 容器也可以根据需要修改自身的容量和大小。

和 vector 不同的是，deque 还==擅长在序列头部添加或删除元素==，所耗费的时间复杂度也为常数阶O(1)。并且更重要的一点是，deque 容器中存储元素并不能保证所有元素都存储到连续的内存空间中。

当需要向==序列两端频繁的添加或删除元素==时，应首选 deque 容器。

| 函数成员            | 函数功能                                                     |
| ------------------- | ------------------------------------------------------------ |
| begin()             | 返回指向容器中第一个元素的迭代器。                           |
| end()               | 返回指向容器最后一个元素所在位置后一个位置的迭代器，通常和 begin() 结合使用。 |
| rbegin()            | 返回指向最后一个元素的迭代器。                               |
| rend()              | 返回指向第一个元素所在位置前一个位置的迭代器。               |
| cbegin()            | 和 begin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| cend()              | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crbegin()           | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crend()             | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| size()              | 返回实际元素个数。                                           |
| max_size()          | 返回容器所能容纳元素个数的最大值。这通常是一个很大的值，一般是 232-1，我们很少会用到这个函数。 |
| resize()            | 改变实际元素的个数。                                         |
| empty()             | 判断容器中是否有元素，若无元素，则返回 true；反之，返回 false。 |
| shrink _to_fit()    | 将内存减少到等于当前元素实际所使用的大小。                   |
| at()                | 使用经过边界检查的索引访问元素。                             |
| front()             | 返回第一个元素的引用。                                       |
| back()              | 返回最后一个元素的引用。                                     |
| assign()            | 用新元素替换原有内容。                                       |
| push_back()         | 在序列的尾部添加一个元素。                                   |
| ==push_front()==    | 在序列的头部添加一个元素。                                   |
| pop_back()          | 移除容器尾部的元素。                                         |
| pop_front()         | 移除容器头部的元素。                                         |
| insert()            | 在指定的位置插入一个或多个元素。                             |
| erase()             | 移除一个元素或一段元素。                                     |
| clear()             | 移出所有的元素，容器大小变为 0。                             |
| swap()              | 交换两个容器的所有元素。                                     |
| emplace()           | 在指定的位置直接生成一个元素。                               |
| ==emplace_front()== | 在容器头部生成一个元素。和 push_front() 的区别是，该函数直接在容器头部构造元素，省去了复制移动元素的过程。 |
| emplace_back()      | 在容器尾部生成一个元素。和 push_back() 的区别是，该函数直接在容器尾部构造元素，省去了复制移动元素的过程。 |

### list链表

**[STL](https://haicoder.net/stl/stl-tutorial.html)** 中的 list 容器，又称双向链表容器，即该容器的底层是以双向链表的形式实现的。这意味着，list 容器中的元素可以分散存储在内存空间里，而不是必须存储在一整块连续的内存空间中。

list 容器中各个元素的前后顺序是靠指针来维系的，每个元素都配备了 2 个指针，分别指向它的前一个元素和后一个元素。其中第一个元素的前向指针总为 null，因为它前面没有元素；同样，尾部元素的后向指针也总为 null。

基于这样的存储结构，list 容器具有一些其它容器（**[array](https://haicoder.net/stl/stl-array.html)**、**[vector](https://haicoder.net/stl/stl-vector.html)** 和 list）所不具备的优势，==即它可以在序列已知的任何位置快速插入或删除元素==（时间复杂度为O(1)）。并且在 list 容器中移动元素，也比其它容器的效率高。

使用 list 容器的缺点是，==它不能像 array 和 vector 那样，通过位置直接访问元素==。举个例子，如果要访问 list 容器中的第 3 个元素，==它不支持容器对象名[3]这种语法格式==，正确的做法是从容器中第一个元素或最后一个元素开始遍历容器，直到找到该位置。

实际场景中，==如何需要对序列进行大量添加或删除元素的操作==，而直接访问元素的需求却很少，这种情况建议使用 list 容器存储序列。

| 成员函数        | 功能                                                         |
| --------------- | ------------------------------------------------------------ |
| begin()         | 返回指向容器中第一个元素的双向迭代器。                       |
| end()           | 返回指向容器中最后一个元素的双向迭代器。                     |
| rbegin()        | 返回指向最后一个元素的反向双向迭代器。                       |
| rend()          | 返回指向第一个元素所在位置前一个位置的反向双向迭代器。       |
| cbegin()        | 和 begin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| cend()          | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crbegin()       | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| crend()         | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改元素。 |
| empty()         | 判断容器中是否有元素，若无元素，则返回 true；反之，返回 false。 |
| size()          | 返回当前容器实际包含的元素个数。                             |
| max_size()      | 返回容器所能包含元素个数的最大值。这通常是一个很大的值，一般是 232-1，所以我们很少会用到这个函数。 |
| front()         | 返回第一个元素的引用。                                       |
| back()          | 返回最后一个元素的引用。                                     |
| assign()        | 用新元素替换容器中原有内容。                                 |
| emplace_front() | 在容器头部生成一个元素。该函数和 push_front() 的功能相同，但效率更高。 |
| push_front()    | 在容器头部插入一个元素。                                     |
| pop_front()     | 删除容器头部的一个元素。                                     |
| emplace_back()  | 在容器尾部直接生成一个元素。该函数和 push_back() 的功能相同，但效率更高。 |
| push_back()     | 在容器尾部插入一个元素。                                     |
| pop_back()      | 删除容器尾部的一个元素。                                     |
| emplace()       | 在容器中的指定位置插入元素。该函数和 insert() 功能相同，但效率更高。 |
| insert()        | 在容器中的指定位置插入元素。                                 |
| erase()         | 删除容器中一个或某区域内的元素。                             |
| swap()          | 交换两个容器中的元素，必须保证这两个容器中存储的元素类型是相同的。 |
| resize()        | 调整容器的大小。                                             |
| clear()         | 删除容器存储的所有元素。                                     |
| splice()        | 将一个 list 容器中的元素插入到另一个容器的指定位置。         |
| remove(val)     | 删除容器中所有等于 val 的元素。                              |
| remove_if()     | 删除容器中满足条件的元素。                                   |
| unique()        | 删除容器中相邻的重复元素，只保留一个。                       |
| merge()         | 合并两个事先已排好序的 list 容器，并且合并之后的 list 容器依然是有序的。 |
| sort()          | 通过更改容器中元素的位置，将它们进行排序。                   |
| reverse()       | 反转容器中元素的顺序。                                       |

## STL关联式容器

**STL** 中的关联式容器主要包括 **[map](https://haicoder.net/stl/stl-map.html)**、**[multimap](https://haicoder.net/stl/stl-multimap.html)**、**[set](https://haicoder.net/stl/stl-set.html)** 以及 **[multiset](https://haicoder.net/stl/stl-multiset.html)**。和 **[序列式容器](https://haicoder.net/stl/stl-sequential-container.html)** 不同的是，关联式容器在存储元素时还会为每个元素在配备一个键，整体以键值对的方式存储到容器中。

相比序列式容器，关联式容器可以通过键值直接找到对应的元素，而无需遍历整个容器。另外，关联式容器在存储元素，默认会根据各元素==键值的大小做升序排序==。相比其它类型容器，关联式容器==查找、访问、插入和删除指定元素的效率更高==。

### pair

C++ **[STL](https://haicoder.net/stl/stl-tutorial.html)** 标准库提供了 pair 类模板，其专门用来将 2 个普通元素 first 和 second（可以是 **[C++](https://haicoder.net/cpp/cpp-tutorial.html)** 基本数据类型、结构体、类自定的类型）创建成一个新元素 `<first, second>`。

通过其构成的元素格式不难看出，使用 pair 类模板来创建 “键值对” 形式的元素，再合适不过。

pair初始化

- 使用花括号

```c++
	#include <utility>

int main() {
    std::pair<int, double> myPair = {42, 3.14};
    // 或者可以使用构造函数的简化语法
    // std::pair<int, double> myPair{42, 3.14};

    // 使用 myPair 进行操作...

    return 0;				
}
```

- 使用构造函数

```c++
#include <utility>

int main() {
    std::pair<int, double> myPair(42, 3.14);

    // 使用 myPair 进行操作...

    return 0;
}
```

- 使用make_pair

```c++
#include <utility>

int main() {
    std::pair<int, double> myPair = std::make_pair(42, 3.14);

    // 使用 myPair 进行操作...

    return 0;
}
```

通过pair.first和pair.second分别访问K和V

### map

map 做为 **[关联容器](https://haicoder.net/stl/stl-associative-container.html)** 的一种，存储的都是 **[pair](https://haicoder.net/stl/stl-pair.html)** 对象，也就是用 pair 类模板创建的键值对。其中，各个键值对的键和值可以是任意数据类型，包括 **[C++](https://haicoder.net/cpp/cpp-tutorial.html)** 基本数据类型（int、double 等）、使用结构体或类自定义的类型。

通常情况下，map 容器中存储的各个键值对都选用 string 字符串作为键的类型。

与此同时，在使用 map 容器存储多个键值对时，该容器会自动根据各键值对的键的大小，按照既定的规则进行排序。默认情况下，map 容器选用 `std::less<T>` 排序规则（其中 T 表示键的数据类型），其会根据键的大小对所有键值对做升序排序。

当然，根据实际情况的需要，我们可以手动指定 map 容器的排序规则，既可以选用 **[STL](https://haicoder.net/stl/stl-tutorial.html)** 标准库中提供的其它排序规则（比如 `std::greater<T>`），也可以自定义排序规则。

另外需要注意的是，使用 map 容器存储的各个键值对，==键的值既不能重复也不能被修改==。换句话说，map 容器中存储的各个键值对不仅键的值独一无二，键的类型也会用 const 修饰，这意味着只要键值对被存储到 map 容器中，其键的值将不能再做任何修改。

map 容器存储的都是 pair 类型的键值对元素，更确切的说，该容器存储的都是 pair<const K, T> 类型（其中 K 和 T 分别表示键和值的数据类型）的键值对元素。

map初始化

- 使用初始化列表：

![image-20231101100755411](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202311011007188.png)

- 使用迭代器范围初始化：

![image-20231101100731164](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202311011007253.png)

- 使用插入操作：

```cpp
#include <map>

int main() {
    std::map<int, std::string> myMap;

    myMap.insert(std::make_pair(1, "one"));
    myMap.insert(std::make_pair(2, "two"));
    myMap.insert(std::make_pair(3, "three"));

    // 使用 myMap 进行操作...

    return 0;
}
```

| begin()          | 返回指向容器中第一个（注意，是已排好序的第一个）键值对的双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| ---------------- | ------------------------------------------------------------ |
| end()            | 返回指向容器最后一个元素（注意，是已排好序的最后一个）所在位置后一个位置的双向迭代器，通常和 begin() 结合使用。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| rbegin()         | 返回指向最后一个（注意，是已排好序的最后一个）元素的反向双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的反向双向迭代器。 |
| rend()           | 返回指向第一个（注意，是已排好序的第一个）元素所在位置前一个位置的反向双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的反向双向迭代器。 |
| cbegin()         | 和 begin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的键值对。 |
| cend()           | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的键值对。 |
| crbegin()        | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的键值对。 |
| crend()          | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的键值对。 |
| find(key)        | 在 map 容器中查找键为 key 的键值对，如果成功找到，则返回指向该键值对的双向迭代器；反之，则返回和 end() 方法一样的迭代器。另外，如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| lower_bound(key) | 返回一个指向当前 map 容器中第一个大于或等于 key 的键值对的双向迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| upper_bound(key) | 返回一个指向当前 map 容器中第一个大于 key 的键值对的迭代器。如果 map 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| equal_range(key) | 该方法返回一个 pair 对象（包含 2 个双向迭代器），其中 pair.first 和 lower_bound() 方法的返回值等价，pair.second 和 upper_bound() 方法的返回值等价。也就是说，该方法将返回一个范围，该范围中包含的键为 key 的键值对（map 容器键值对唯一，因此该范围最多包含一个键值对）。 |
| empty()          | 若容器为空，则返回 true；否则 false。                        |
| size()           | 返回当前 map 容器中存有键值对的个数。                        |
| max_size()       | 返回 map 容器所能容纳键值对的最大个数，不同的操作系统，其返回值亦不相同。 |
| operator[]       | map容器重载了 [] 运算符，只要知道 map 容器中某个键值对的键的值，就可以向获取数组中元素那样，通过键直接获取对应的值。 |
| at(key)          | 找到 map 容器中 key 键对应的值，如果找不到，该函数会引发 out_of_range 异常。 |
| insert()         | 向 map 容器中插入键值对。                                    |
| erase()          | 删除 map 容器指定位置、指定键（key）值或者指定区域内的键值对。后续章节还会对该方法做重点讲解。 |
| swap()           | 交换 2 个 map 容器中存储的键值对，这意味着，操作的 2 个键值对的类型必须相同。 |
| clear()          | 清空 map 容器中所有的键值对，即使 map 容器的 size() 为 0。   |
| emplace()        | 在当前 map 容器中的指定位置处构造新键值对。其效果和插入键值对一样，但效率更高。 |
| emplace_hint()   | 在本质上和 emplace() 在 map 容器中构造新键值对的方式是一样的，不同之处在于，使用者必须为该方法提供一个指示键值对生成位置的迭代器，并作为该方法的第一个参数。 |
| count(key)       | 在当前 map 容器中，查找键为 key 的键值对的个数并返回。注意，由于 map 容器中各键值对的键的值是唯一的，因此该函数的返回值最大为 1。 |

### set

**[STL](https://haicoder.net/stl/stl-tutorial.html)** 中的 set 也是一种 **[关联式容器](https://haicoder.net/stl/stl-associative-container.html)**，其用来存储一组不重复的元素集合，而且，set 存储元素和 **[map](https://haicoder.net/stl/stl-map.html)** 以及 **[multimap](https://haicoder.net/stl/stl-multimap.html)** 类似，==都是有序的==。

从语法上讲 set 容器并没有强制对存储元素的类型做 const 修饰，即 set 容器中存储的元素的值是可以修改的。但是，**[C++](https://haicoder.net/cpp/cpp-tutorial.html)** 标准为了防止用户修改容器中元素的值，对所有可能会实现此操作的行为做了限制，使得在正常情况下，用户是无法做到修改 set 容器中元素的值的。

对于初学者来说，切勿尝试直接修改 set 容器中已存储元素的值，这很有可能破坏 set 容器中元素的有序性，最正确的修改 set 容器中元素值的做法是：先删除该元素，然后再添加一个修改后的元素。

set初始化

```c++
#使用初始化列表
std::set<int> mySet = {1, 2, 3, 4, 5};

#使用插入初始化
std::set<int> mySet;
mySet.insert(1);
mySet.insert(2);
mySet.insert(3);
mySet.insert(4);
mySet.insert(5);
```

| 成员方法         | 功能                                                         |
| ---------------- | ------------------------------------------------------------ |
| begin()          | 返回指向容器中第一个（注意，是已排好序的第一个）元素的双向迭代器。如果 set 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| end()            | 返回指向容器最后一个元素（注意，是已排好序的最后一个）所在位置后一个位置的双向迭代器，通常和 begin() 结合使用。如果 set 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| rbegin()         | 返回指向最后一个（注意，是已排好序的最后一个）元素的反向双向迭代器。如果 set 容器用 const 限定，则该方法返回的是 const 类型的反向双向迭代器。 |
| rend()           | 返回指向第一个（注意，是已排好序的第一个）元素所在位置前一个位置的反向双向迭代器。如果 set 容器用 const 限定，则该方法返回的是 const 类型的反向双向迭代器。 |
| cbegin()         | 和 begin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的元素值。 |
| cend()           | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的元素值。 |
| crbegin()        | 和 rbegin() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的元素值。 |
| crend()          | 和 rend() 功能相同，只不过在其基础上，增加了 const 属性，不能用于修改容器内存储的元素值。 |
| find(val)        | 在 set 容器中查找值为 val 的元素，如果成功找到，则返回指向该元素的双向迭代器；反之，则返回和 end() 方法一样的迭代器。另外，如果 set 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| lower_bound(val) | 返回一个指向当前 set 容器中第一个大于或等于 val 的元素的双向迭代器。如果 set 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| upper_bound(val) | 返回一个指向当前 set 容器中第一个大于 val 的元素的迭代器。如果 set 容器用 const 限定，则该方法返回的是 const 类型的双向迭代器。 |
| equal_range(val) | 该方法返回一个 pair 对象（包含 2 个双向迭代器），其中 pair.first 和 lower_bound() 方法的返回值等价，pair.second 和 upper_bound() 方法的返回值等价。也就是说，该方法将返回一个范围，该范围中包含的值为 val 的元素（set 容器中各个元素是唯一的，因此该范围最多包含一个元素）。 |
| empty()          | 若容器为空，则返回 true；否则 false。                        |
| size()           | 返回当前 set 容器中存有元素的个数。                          |
| max_size()       | 返回 set 容器所能容纳元素的最大个数，不同的操作系统，其返回值亦不相同。 |
| insert()         | 向 set 容器中插入元素。                                      |
| erase()          | 删除 set 容器中存储的元素。                                  |
| swap()           | 交换 2 个 set 容器中存储的所有元素。这意味着，操作的 2 个 set 容器的类型必须相同。 |
| clear()          | 清空 set 容器中所有的元素，即令 set 容器的 size() 为 0。     |
| emplace()        | 在当前 set 容器中的指定位置直接构造新元素。其效果和 insert() 一样，但效率更高。 |
| emplace_hint()   | 在本质上和 emplace() 在 set 容器中构造新元素的方式是一样的，不同之处在于，使用者必须为该方法提供一个指示新元素生成位置的迭代器，并作为该方法的第一个参数。 |
| count(val)       | 在当前 set 容器中，查找值为 val 的元素的个数，并返回。注意，由于 set 容器中各元素的值是唯一的，因此该函数的返回值最大为 1。 |

## STL无序关联式容器

### unordered_map

unordered_map 容器，直译过来就是 “无序 map 容器” 的意思。所谓 “无序”，指的是 unordered_map 容器不会像 **[map](https://haicoder.net/stl/stl-map.html)** 容器那样对存储的数据进行排序。

换句话说，unordered_map 容器和 map 容器仅有一点不同，即 map 容器中存储的数据是有序的，而 unordered_map 容器中是无序的。

具体来讲，unordered_map 容器和 map 容器一样，以键值对，即 **[pair 类型](https://haicoder.net/stl/stl-pair.html)** 的形式存储数据，存储的各个键值对的键互不相同且不允许被修改。

但由于 unordered_map 容器底层采用的是哈希表存储结构，该结构本身不具有对数据的排序功能，所以此容器内部不会自行对存储的键值对进行排序。

| 成员方法           | 功能                                                         |
| ------------------ | ------------------------------------------------------------ |
| begin()            | 返回指向容器中第一个键值对的正向迭代器。                     |
| end()              | 返回指向容器中最后一个键值对之后位置的正向迭代器。           |
| cbegin()           | 和 begin() 功能相同，只不过在其基础上增加了 const 属性，即该方法返回的迭代器不能用于修改容器内存储的键值对。 |
| cend()             | 和 end() 功能相同，只不过在其基础上，增加了 const 属性，即该方法返回的迭代器不能用于修改容器内存储的键值对。 |
| empty()            | 若容器为空，则返回 true；否则 false。                        |
| size()             | 返回当前容器中存有键值对的个数。                             |
| max_size()         | 返回容器所能容纳键值对的最大个数，不同的操作系统，其返回值亦不相同。 |
| operator[key]      | 该模板类中重载了 [] 运算符，其功能是可以向访问数组中元素那样，只要给定某个键值对的键 key，就可以获取该键对应的值。注意，如果当前容器中没有以 key 为键的键值对，则其会使用该键向当前容器中插入一个新键值对。 |
| at(key)            | 返回容器中存储的键 key 对应的值，如果 key 不存在，则会抛出 out_of_range 异常。 |
| find(key)          | 查找以 key 为键的键值对，如果找到，则返回一个指向该键值对的正向迭代器；反之，则返回一个指向容器中最后一个键值对之后位置的迭代器（如果 end() 方法返回的迭代器）。 |
| count(key)         | 在容器中查找以 key 键的键值对的个数。                        |
| equal_range(key)   | 返回一个 pair 对象，其包含 2 个迭代器，用于表明当前容器中键为 key 的键值对所在的范围。 |
| emplace()          | 向容器中添加新键值对，效率比 insert() 方法高。               |
| emplace_hint()     | 向容器中添加新键值对，效率比 insert() 方法高。               |
| insert()           | 向容器中添加新键值对。                                       |
| erase()            | 删除指定键值对。                                             |
| clear()            | 清空容器，即删除容器中存储的所有键值对。                     |
| swap()             | 交换 2 个 unordered_map 容器存储的键值对，前提是必须保证这 2 个容器的类型完全相等。 |
| bucket_count()     | 返回当前容器底层存储键值对时，使用桶（一个线性链表代表一个桶）的数量。 |
| max_bucket_count() | 返回当前系统中，unordered_map 容器底层最多可以使用多少桶。   |
| bucket_size(n)     | 返回第 n 个桶中存储键值对的数量。                            |
| bucket(key)        | 返回以 key 为键的键值对所在桶的编号。                        |
| load_factor()      | 返回 unordered_map 容器中当前的负载因子。负载因子，指的是的当前容器中存储键值对的数量（size()）和使用桶数（bucket_count()）的比值，即 load_factor() = size() / bucket_count()。 |
| max_load_factor()  | 返回或者设置当前 unordered_map 容器的负载因子。              |
| rehash(n)          | 将当前容器底层使用桶的数量设置为 n。                         |
| reserve()          | 将存储桶的数量（也就是 bucket_count() 方法的返回值）设置为至少容纳count个元（不超过最大负载因子）所需的数量，并重新整理容器。 |
| hash_function()    | 返回当前容器使用的哈希函数对象。                             |

### unordered_set

unordered_set 容器，可直译为 “无序 set 容器”，即 unordered_set 容器和 set 容器很像，唯一的区别就在于 set 容器会自行对存储的数据进行排序，而 unordered_set 容器不会。

STL unordered_set特征

1. 不再以键值对的形式存储数据，而是直接存储数据的值。
2. 容器内部存储的各个元素的值都互不相等，且不能被修改。
3. 不会对内部存储的数据进行排序。

| 成员方法           | 功能                                                         |
| ------------------ | ------------------------------------------------------------ |
| begin()            | 返回指向容器中第一个元素的正向迭代器。                       |
| end();             | 返回指向容器中最后一个元素之后位置的正向迭代器。             |
| cbegin()           | 和 begin() 功能相同，只不过其返回的是 const 类型的正向迭代器。 |
| cend()             | 和 end() 功能相同，只不过其返回的是 const 类型的正向迭代器。 |
| empty()            | 若容器为空，则返回 true；否则 false。                        |
| size()             | 返回当前容器中存有元素的个数。                               |
| max_size()         | 返回容器所能容纳元素的最大个数，不同的操作系统，其返回值亦不相同。 |
| find(key)          | 查找以值为 key 的元素，如果找到，则返回一个指向该元素的正向迭代器；反之，则返回一个指向容器中最后一个元素之后位置的迭代器（如果 end() 方法返回的迭代器）。 |
| count(key)         | 在容器中查找值为 key 的元素的个数。                          |
| equal_range(key)   | 返回一个 pair 对象，其包含 2 个迭代器，用于表明当前容器中值为 key 的元素所在的范围。 |
| emplace()          | 向容器中添加新元素，效率比 insert() 方法高。                 |
| emplace_hint()     | 向容器中添加新元素，效率比 insert() 方法高。                 |
| insert()           | 向容器中添加新元素。                                         |
| erase()            | 删除指定元素。                                               |
| clear()            | 清空容器，即删除容器中存储的所有元素。                       |
| swap()             | 交换 2 个 unordered_map 容器存储的元素，前提是必须保证这 2 个容器的类型完全相等。 |
| bucket_count()     | 返回当前容器底层存储元素时，使用桶（一个线性链表代表一个桶）的数量。 |
| max_bucket_count() | 返回当前系统中，unordered_map 容器底层最多可以使用多少桶。   |
| bucket_size(n)     | 返回第 n 个桶中存储元素的数量。                              |
| bucket(key)        | 返回值为 key 的元素所在桶的编号。                            |
| load_factor()      | 返回 unordered_map 容器中当前的负载因子。负载因子，指的是的当前容器中存储元素的数量（size()）和使用桶数（bucket_count()）的比值，即 load_factor() = size() / bucket_count()。 |
| max_load_factor()  | 返回或者设置当前 unordered_map 容器的负载因子。              |
| rehash(n)          | 将当前容器底层使用桶的数量设置为 n。                         |
| reserve()          | 将存储桶的数量（也就是 bucket_count() 方法的返回值）设置为至少容纳count个元（不超过最大负载因子）所需的数量，并重新整理容器。 |
| hash_function()    | 返回当前容器使用的哈希函数对象。                             |

## 容器的常见操作

### 排序

要对 `std::vector` 进行排序，可以使用标准库提供的排序算法 `std::sort`。`std::sort` 函数可以按升序对容器中的元素进行排序。以下是对 `std::vector` 进行排序的示例代码：

```C++
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> myVector = {5, 2, 3, 1, 4};

    // 使用 std::sort 对容器进行排序
    std::sort(myVector.begin(), myVector.end());

    // 输出排序后的结果
    for (const auto& element : myVector) {
        std::cout << element << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

### 遍历

迭代器遍历unordered_map，迭代器是一个类似指针的对象。在 C++ 中，迭代器是一种抽象的概念，它提供了一组操作符（如 `*` 和 `++`）来访问容器中的元素。迭代器可以被视为容器内部的指针，它指向容器中的某个元素或位置。

对于 `std::unordered_map` 容器，`myMap.begin()` 返回的是一个迭代器，指向容器中第一个键值对的位置。通过解引用迭代器，可以获取该位置上的键值对。

![image-20231101100822192](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202311011008487.png)

遍历vector容器，通过`* ` 解引用

```c++
#include <iostream>
#include <vector>

int main() {
    std::vector<int> myVector = {1, 2, 3, 4, 5};

    // 使用迭代器遍历容器
    for (auto it = myVector.begin(); it != myVector.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;

    return 0;
}
```










