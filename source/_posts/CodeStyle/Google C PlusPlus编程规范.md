---
title: Google C++ 编程规范
date: 2020-11-06 11：24：12
categories: codeing
tags:
	- code style
	- C++
---



# 头文件

1. 头文件包含顺序，本类的头文件、C库、C++库、其他的.h库，项目内的.h库。
2. 项目内头文件推荐按照字典序排序。
3. 头文件的包含推荐使用其在项目中的完整路径。

# 作用域

## 命名空间

### 不具名命名空间

就是namespace不加名字，在.cc文件中，允许甚至提倡使用不具名命名空间，以避免运行时的命名冲突：

```c++
namespace { // .cc 文件中
// 命名空间的内容无需缩进
enum { UNUSED, EOF, ERROR }; // 经常使用的符号
bool AtEof() { return pos_ == EOF; } // 使用本命名空间内的符号EOF
} // namespace
```

不能再.h文件中使用不具名命名空间。

### 具名命名空间

具名命名空间的格式为：namespace xxxname{}

命名空间之前包含：

1. 文件包含
2. 全局标识的声明/定义
3. 类的前置声明

```c++
// .h 文件
namespace mynamespace {
// 所有声明都置于命名空间中
// 注意不要使用缩进
class MyClass {
public:
...
void Foo();
};
} // namespace mynamespace
// .cc 文件
namespace mynamespace {
// 函数定义都置于命名空间中
void MyClass::Foo() {
...
}
} // namespace mynamespace
```

还应该注意的点是：

1. 不要声明命名空间std下的任何内容，包括标准库类的前置声明。声明std下的实体会导致不明确的行为，如，不可移植。

2. 声明标准库下的实体，需要包含对应的头文件。

3. 最好不要使用using指示符，以保证命名空间下的所有名称都正常使用。

   ```c++
   // 禁止 污染命名空间
   using namespace foo;
   
   // 在.cc文件，.h文件的函数、方法或者类中，可以使用Using
   // 允许：.cc 文件中
   // .h 文件中，必须在函数、方法或类的内部使用
   using ::foo::bar;
   
   // 在.cc 文件、.h 文件的函数、方法或类中，还可以使用命名空间别名。
   // 允许：.cc 文件中
   // .h 文件中，必须在函数、方法或类的内部使用
   namespace fbz = ::foo::bar::baz;
   ```

## 非成员函数、静态成员函数和全局函数

使用命名空间内的非成员函数或静态成员函数，尽量不要使用全局函数。



# 局部变量

1. 将函数变量尽可能置于最小作用域内，在声明变量时将其初始化。

2. 尽可能在小的作用域中声明变量，离第一次越近越好。

3. 应使用初始化代替声明+赋值的方式。

   ```c++
   int i;
   i = f(); // 坏——初始化和声明分离
   nt j = g(); // 好——初始化时声明
   ```

4. 如果变量是一个对象，每次进入作用域要调用其构造函数，退出作用域要调用其析构函数。

   ```c++
   // 低效的实现
   for (int i = 0; i < 1000000; ++i) {
   	Foo f; // 构造函数和析构函数分别调用1000000 次！
   	f.DoSomething(i);
   }
   
   Foo f; // 构造函数和析构函数只调用1 次
   for (int i = 0; i < 1000000; ++i) {
   	f.DoSomething(i);
   }
   ```

   

## 全局变量

1. class 类型的全局变量是被禁止的，内建类型的全局变量是允许的，当然多线程代码中非常数全局变量也是被禁止的。永远不要使用函数返回值初始化全局变量。

2. 对于全局的字符串常量，使用C 风格的字符串，而不要使用STL 的字符串。

   ```c++
   const char kFrogSays[] = "ribbet";
   ```

3. 静态成员变量视作全局变量，所以，也不能是class 类型。

# 类

1. 构造函数中只进行简单的初始化，，可能的话，使用Init()方法集中初始化为有意义的（non-trivial）数据。

2. 构造函数内调用虚函数，调用不会派发到子类实现中，即使当前没有子类化实现，将来仍是隐患。

3. 如果构建了全局类型的class，则该类的构造函数将在main()之前被调用。

4. 如果对象需要有意义的（non-trivial）初始化，考虑使用另外的Init()方法并（或）增加一个成员标记用于指示对象是否已经初始化成功。

5. 如果类中定义了成员变量，没有提供其他构造函数，你需要定义一个默认构造函数（没有参数）。默认构造函数更适合于初始化对象，使对象内部状态（internal state）一致、有效。

6. 对单参数构造函数使用C++关键字explicit。

7. 不需要拷贝时应使用DISALLOW_COPY_AND_ASSIGN，仅在代码中需要拷贝一个类对象的时候使用拷贝构造函数。

   ```c++
   // 禁止使用拷贝构造函数和赋值操作的宏
   // 应在类的private:中使用
   #define DISALLOW_COPY_AND_ASSIGN(TypeName) \
   	TypeName(const TypeName&); \
   	void operator=(const TypeName&)
   class Foo {
   public:
   	Foo(int f);
   	~Foo();
   private:
   	DISALLOW_COPY_AND_ASSIGN(Foo);
   };
   ```

8. 在将类作为STL 容器值得时候，你可能有使类可拷贝的冲动。类似情况下，真正该做的是使用指针指向STL 容器中的对象，可以考虑使用std::tr1::shared_ptr。

9. struct 被用在仅包含数据的消极对象（passive objects）上，可能包括有关联的常量，但没有存取数据成员之外的函数功能，而存取功能通过直接访问实现而无需方法调用，这儿提到的方法是指只用于处理数据成员的，如构造函数、析构函数、Initialize()、Reset()、Validate()。

10. 如果需要更多的函数功能，class 更适合，如果不确定的话，直接使用class。

11. 如果与STL 结合，对于仿函数（functors）和特性（ traits）可以不用class 而是使用struct。

12. 所有继承必须是public 的，如果想私有继承的话，应该采取包含基类实例作为成员的方式作为替代。

13. 在含有虚析构函数的父类中，定义虚构函数绝对必要。

14. 限定仅在子类访问的成员函数为protected，需要注意的是数据成员应始终为私有。当重定义派生的虚函时，在派生类中明确声明其为virtual。根本原因：如果遗漏virtual，阅读者需要检索类的所有祖先以确定该函数否为虚函数（译者注，虽然不影响其为虚函数的本质）

15. 一般不要重载操作符，尤其是赋值操作（operator=）比较阴险，应避免重载。如果需要的话，可以定义类似Equals()、CopyFrom()等函数。

16. 函数长度在40行内比较好，但是对于逻辑比较复杂的函数，没有这个要求。



# 变量声明顺序

1. public
2. protected
3. private
4. 没一个块中，声明次序如下：
   - typdefs和enums
   - 常量
   - 构造函数
   - 析构函数
   - 成员函数，含静态成员函数
   - 数据成员，含静态数据成员
5. 宏`DISALLOW_COPY_AND_ASSIGN`置于private块之后，作为类的最后部分，参考拷贝构造函数
6. .cc文件中，函数的定义应尽可能和声明次序一致。

# 类型转换

1. static_cast：和C 风格转换相似可做值的强制转换，或指针的父类到子类的明确的向上转换；
2. const_cast：移除const 属性；
3. reinterpret_cast：指针类型和整型或其他指针间不安全的相互转换，仅在你对所做一切了然于心时使用；
4. dynamic_cast：除测试外不要使用，除单元测试外，如果你需要在运行时确定类型信息，说明设计有缺陷（参考RTTI）

# Google特有的风情

1. 不推荐使用智能指针；
2. 可以使用scoped_ptr代替智能指针，使用时尽量局部化，并且，安全第一；
3. **事实上这是一个硬性约定：输入参数为值或常数引用，输出参数为指针；输入参数可以是常数指针，但不能使用非常数引用形参。
   在强调参数不是拷贝而来，在对象生命期内必须一直存在时可以使用常数指针，最好将这些在注释中详细说明。bind2nd 和mem_fun 等STL 适配器不接受引用形参，这种情况下也必须以指针形参声明函数。**
4. 所有参数必须明确指定，强制程序员考虑API 和传入的各参数值，避免使用可能不为程序员所知的缺省参数。
5. 禁止使用异常。

# 其他C++特性

## 流

1. 只在记录日志时使用流；
2. 流是printf和scanf的替代；
3. 使用流还有很多利弊，代码一致性胜过一切，不要在代码中使用流；
4. 最后的多数决定是printf + read/write

## 前置自增和自减

1. 对于迭代器和其他模板对象使用前缀形式（++i）的自增、自减运算符；
2. 不考虑返回值的话，前置自增（++i）通常要比后置自增（i++）效率更高，因为后置的自增自减需要对表达式的值i 进行一次拷贝，如果i 是迭代器或其他非数值类型，拷贝的代价是比较大的。既然两种自增方式动作一样（译者注，不考虑表达式的值，相信你知道我在说什么），为什么不直接使用前置自增呢。
3. 对简单数值（非对象）来说，两种都无所谓，对迭代器和模板类型来说，要使用前置自增（自减）；

##　Const的使用

1. 在声明的变量或参数前加上关键字const 用于指明变量值不可修改（如const intfoo），为类中的函数加上const 限定表明该函数不会修改类成员变量的状态（如class Foo{ int Bar(char c) const; };）。
2. 在以下情况下可以考虑使用const
   - 如果函数不会修改传入的引用或指针类型的参数，这样的参数应该为const；
   - 尽可能将函数声明为const，访问韩式应该总是const，其他函数如果不会修改任何数据成员也应该是const，不要调用非const函数，不要返回对数据成员的非const指针或引用；
   - 如果数据成员在对象构造之后不再改变，可将其定义为const。

## 整型

1. 提倡使用精确宽度的整形替代int，因为int在不同类型的操作系统中其位数是不同的。，通常人们认为short 是16 位，int 是32 位，long 是32位，long long 是64 位。

2. <stdint.h>定义了int16_t、uint32_t、int64_t 等整型，在需要确定大小的整型时可以使用它们代替short、unsigned long long 等；，在C 整型中，只使用int。适当情况下，推荐使用标准类型如size_t 和ptrdiff_t。

3. 不要使用uint32_t 等无符号整型，除非你是在表示一个位组（bit pattern）而不是一个数值。即使数值不会为负值也不要使用无符号类型。使用断言来保护数据。

4. 在C 语言中，无符号整形通常会导致Bug。如，以下情况会导致死循环，原因是，当i减到0时，由于它是无符号整数，当其变为-1时，它的二进制形式为32个1，在判断i>=0时，会将其看为2^31 - 1，而不是-1，从而导致死循环。

   ```c
   #include <stdio.h>
   
   int main() {
   
       for (unsigned int i = 100; i >= 0; i--) {
           printf("%d\n", i); 
       }   
       return 0;
   }
   
   /* 输出如下：
   ......
   8
   7
   6
   5
   4
   3
   2
   1
   0
   -1
   -2
   -3
   -4
   -5
   -6
   ......
   */
   ```

5. 打印整型

   ```c++
   //TODO
   ```

6. sizeof(void *) != sizeof(int)，，如果需要一个指针大小的整数要使用intptr_t

   ```c++
   int main() {
       printf("%d\n", sizeof(void*));
       printf("%d\n", sizeof(int));
       return 0;
   }
   
   /* 输出
   * 8 // 指针
   * 4 // int
   /
   ```

   

7. 需要对结构对齐加以留心，尤其是对于存储在磁盘上的结构体。在64 位系统中，任何拥有int64_t/uint64_t 成员的类/结构体将默认被处理为8 字节对齐。如果32 位和64 位代码共用磁盘上的结构体，需要确保两种体系结构下的结构体的对齐一致。大多数编译器提供了调整结构体对齐的方案。gcc 中可使用__attribute__((packed))，MSVC 提供了#pragma pack()和__declspec(align())（译者注，解决方案的项目属性里也可以直接设置）

8. 创建64 位常量时使用LL 或ULL 作为后缀

   ```c++
   int64_t my_value = 0x123456789LL;
   uint64_t my_mask = 3ULL << 48;
   ```

## 预处理宏

使用宏时要谨慎，尽量以内联函数，枚举和常量代替。在C++中，宏不像C中那么重要，宏内敛效率关键代码（performance-criticalcode）可以内联函数替代；宏存储常量可以const 变量替代；宏“缩写”长变量名可以引用
替代；

宏可以做一些其他技术无法实现的事情，如字符串化使用#， 连接（concatenation，译者注，使用##）等等，不过在使用前，考虑以下能不能不使用宏实现相同的效果。

1. 不要在.h 文件中定义宏；
2. 使用前正确#define，使用后正确#undef
3. 不要只是对已经存在的宏使用#undef，选择一个不会冲突的名称
4. 不使用会导致不稳定的C++构造（unbalanced C++ constructs，译者注）的宏

## 0 和 NULL

1. 整数用0
2. 实数用0.0
3. 指针用NULL或者nullptr
4. 字符用`\0`

## Sizeof

尽可能用sizeof(varname)代替sizeof(type)

```c++
Struct data;
memset(&data, 0, sizeof(data));
memset(&data, 0, sizeof(Struct));
```



## Boost

只使用Boost中被认可的库。

1. Compressed Pair：boost/compressed_pair.hpp；
2. Pointer Container：boost/ptr_container 不包括ptr_array.hpp 和序列化（serialization）;
3. **Call Traits** from boost/call_traits.hpp;
4. **The Boost Graph Library (BGL)** from boost_graph，除了serializetion(adj_list_serialize.hpp)和parallel/distributed algorithms（boost/graph/parallel）和data structures（boost/graph/distributed/*）
5. **Property Map** from boost/property_map,  除了parallel/distributed property maps(boost/property_map/parallel/*)
6. **Iterator** from boost/iterator
7. **Bimap** from boost/bitmap
8. **Statistical Distributions and Functions** from boost/math/distributions
9. **Multi-index** from boost/multi_index
10. **Heap** from boost/heap
11. **Container** from boost/container/flat_map 和 boost/container/flat_set
12. **Intrusive** from boost/intrusive
13. The part of **Polygon** that deals with Voronoi diagram construction and doesn't depend on the
    rest of Polygon: boost/polygon/voronoi_builder.hpp,boost/polygon/voronoi_diagram.hpp,
    and boost/polygon/voronoi_geometry_type.hpp
14. 下面两个也是允许的，但是它们已经被C++11放弃了
    1. **Array** from boost/array.hpp
    2. **Pointer Container** from boost/ptr_container: use containers of **std::unique_ptr** instead



## Aliases

推荐使用Aliases，因为它能使得API更加清晰，下面有三种Aliases的形式

```c++
typedef Foo Bar;
using Bar = Foo;
using other_namespace::Foo;
```



但是它也有一些缺点，因此在使用时，必须在文件中声明怎么使用这些新的类型。

下面的代码是推荐的：

```c++
namespace a {
// Used to store field measurements. DataPoint may change from Bar* to some internal type.
// Client code should treat it as an opaque pointer.
using DataPoint = foo::bar::Bar*;
// A set of measurements. Just an alias for user convenience.
using TimeSeries = std::unordered_set<DataPoint, std::hash<DataPoint>,
DataPointComparator>;
} // namespace a
```



下面的代码是不推荐的：

```c++
namespace a {
// Bad: none of these say how they should be used.
using DataPoint = foo::bar::Bar*;
using std::unordered_set; // Bad: just for local convenience
using std::hash; // Bad: just for local convenience
typedef unordered_set<DataPoint, hash<DataPoint>, DataPointComparator> TimeSeries;
} // namespace a
```

但是在函数内部、类的私有部分、明确的中间类型的namspace以及在.cc文件中，aliases是推荐的

```c++
// In a .cc file
using std::unordered_set;
```





## Braced Initializer List

直接使用大括号初始化

```c++
// Vector takes a braced-init-list of elements.
vector<string> v{"foo", "bar"};
// Basically the same, ignoring some small technicalities.
// You may choose to use either form.
vector<string> v = {"foo", "bar"};
// Usable with 'new' expressions.
auto p = new vector<string>{"foo", "bar"};
// A map can take a list of pairs. Nested braced-init-lists work.
map<int, string> m = {{1, "one"}, {2, "2"}};
// A braced-init-list can be implicitly converted to a return type.
vector<int> test_function() { return {1, 2, 3}; }
// Iterate over a braced-init-list.
for (int i : {-1, -2, -3}) {}
// Call a function using a braced-init-list.
void TestFunction2(vector<int> v) {}
TestFunction2({1, 2, 3});
```

可以为类定义大括号初始化

```c++
class MyType {
public:
    // std::initializer_list references the underlying init list.
    // It should be passed by value.
    MyType(std::initializer_list<int> init_list) {
        for (int i : init_list) append(i);
    }
	MyType& operator=(std::initializer_list<int> init_list) {
    	clear();
    	for (int i : init_list) append(i);
    }
};
MyType m{2, 3, 5, 7};
```

有些类的构造函数即使没有声明大括号类型的构造函数，也可可以进行转化

```c++
double d{1.23};
// Calls ordinary constructor as long as MyOtherType has no
// std::initializer_list constructor.
class MyOtherType {
public:
    explicit MyOtherType(string);
    MyOtherType(int, string);
};
MyOtherType m = {1, "b"};
// If the constructor is explicit, you can't use the "= {}" form.
MyOtherType m{"b"};
```

不要使用大括号赋值法为auto类型的变量赋值，因为这种是不可控的。

```c++
auto d = {1.23}; // Bad --- d is a std::initializer_list<double>
auto d = double{1.23}; // Good -- d is a double, not a std::initializer_list.
```

## Lambda 表达式

尽可能使用lambda表达式，当函数参数是一个函数时，用lambda表达式比较合适

```c++
std::sort(v.begin(), v.end(), [](int x, int y) {
	return Weight(x) < Weight(y);
});
```

1. 在STL中常用lambda表达式
2. 如果一个lambda表达式的长度超过5行，考虑使用命名的lambda表达式或者函数进行替代。

# 命名约定

## 通用命名规则

1. 函数命名、变量命名、文件命名应具有描述性，不要过度缩写，类型和变量应该是名词，函数名可以用“命令性”动词

2. 尽可能给出描述性名称，不要节约空间，让别人很快理解你的代码更重要，好的命名选择：

   ```c++
   int num_errors; // Good.
   int num_completed_connections; // Good.
   ```

   坏的命名：

   ```c++
   int n; // Bad - meaningless.
   int nerr; // Bad - ambiguous abbreviation.
   int n_comp_conns; // Bad - ambiguous abbreviation.
   ```

   

3. 函数名通常是指令性的，如OpenFile()、set_num_errors()，访问函数需要描述的更细致，要与其访问的变量相吻合.

4. 缩写

   ```c++
   // Good
   // These show proper names with no abbreviations.
   int num_dns_connections; // Most people know what "DNS" stands for.
   int price_count_reader; // OK, price count. Makes sense.
   
   // Bad!
   // Abbreviations can be confusing or ambiguous outside a small group.
   int wgc_connections; // Only your group knows what this stands for.
   int pc_reader; // Lots of things can be abbreviated "pc".
   
   // 不要用省略字母的缩写：
   int error_count; // Good.
   int error_cnt; // Bad.
   ```

   

## 文件命名

1. 文件名要全部小写，可以包含下划线（_）或短线（-），按项目约定来。

```
// 以下都可以
my_useful_class.cc
my-useful-class.cc
myusefulclass.cc
```

2. C++文件以.cc 结尾，头文件以.h 结尾。

3. 不要使用已经存在于/usr/include 下的文件名（译者注，对UNIX、Linux 等系统而言），如db.h；

4. 通常，尽量让文件名更加明确，http_server_logs.h 就比logs.h 要好，定义类时文件名一般成对出现，如foo_bar.h 和foo_bar.cc，对应类FooBar;

5. 内联函数必须放在.h 文件中，如果内联函数比较短，就直接放在.h 中。如果代码比较长，可以放到以-inl.h 结尾的文件中。对于包含大量内联代码的类，可以有三个文件：

   ```c++
   url_table.h // The class declaration.
   url_table.cc // The class definition.
   url_table-inl.h // Inline functions that include lots of code.
   ```

   

## 类型命名

1. 类型命名每个单词以大写字母开头，不包含下划线： MyExcitingClass, MyExcitingEnum
2. 类型命名每个单词以大写字母开头，不包含下划线：MyExcitingClass、MyExcitingEnum。所有类型命名——类、结构体、类型定义（typedef）、枚举——使用相同约定。

## 变量命名

1. 变量名一律小写，单词间以下划线相连，类的成员变量以下划线结尾

   ```c++
   string table_name; // OK - uses underscore.
   string tablename; // OK - all lowercase.
   string tableName; // Bad - mixed case.
   ```

   

2. 结构体的数据成员可以和普通变量一样，不用像类那样接下划线：

   ```c++
   struct UrlTableProperties {
       string name;
       int num_entries;
   }
   ```

3. 对全局变量没有特别要求，少用就好，可以以g_或其他易与局部变量区分的标志为前缀。

4. 常量命名：在名称前加k：kDaysInAWeek。所有编译时常量（无论是局部的、全局的还是类中的）和其他变量保持些许区别，k 后接大写字母开头的单词：

   ```c++
   const int kDaysInAWeek = 7;
   ```

## 函数命名

普通函数大小写混合，存取函数则要求与变量名匹配，MyExcitingFunction()、MyExcitingMethod()、my_exciting_member_variable()。

1. 普通函数:函数名以大写字母开头，每个单词首字母大写，没有下划线：

   ```
   AddTableEntry()
   DeleteUrl()
   ```

   

2. 存取函数要与存取的变量名匹配

   ```c++
   class MyClass {
   public:
   ...
   	int num_entries() const { return num_entries_; }
   	void set_num_entries(int num_entries) { num_entries_ = num_entries; }
   private:
   	int num_entries_;
   };
   ```

3. 其他短小的内联函数名也可以使用小写字母，例如，在循环中调用这样的函数甚至都不需要
   缓存其值，小写命名就可以接受。
4. 从这一点上可以看出，小写的函数名意味着可以直接内联使用。

## 命名空间

1. 命名空间的名称是全小写的，其命名基于项目名称和目录结构：google_awesome_project。

## 枚举命名

1. 枚举值应全部大写，单词间以下划线相连
2. 枚举名称属于类型，因此大小写混合：UrlTableErrors。

```c++
enum UrlTableErrors {
    OK = 0,
    ERROR_OUT_OF_MEMORY,
    ERROR_MALFORMED_INPUT,
};
```



## 宏命名

通常不推荐使用宏，如果使用，其命名像枚举命名一样全部大写使用下划线。

```c++
#define ROUND(x) ...
#define PI_ROUNDED 3.0
MY_EXCITING_ENUM_VALUE
```

# 注释

## 函数注释

1. 函数声明处(.h)注释描述函数功能，定义处(.cc)描述函数实现。

2. 函数声明处注释的内容

   - inputs（输入）及outputs（输出）
   - 对类成员函数而言：函数调用期间对象是否需要保持引用参数，是否会释放这些参数
   - 如果函数分配了空间，需要由调用者释放
   - 参数是否可以为NULL
   - 是否存在函数使用的性能隐忧（performance implications）；
   - 如果函数是可重入的（re-entrant），其同步前提是什么

   ```c++
   // Returns an iterator for this table. It is the client's
   // responsibility to delete the iterator when it is done with it,
   // and it must not use the iterator once the GargantuanTable object
   // on which the iterator was created has been deleted.
   //
   // The iterator is initially positioned at the beginning of the table.
   //
   // This method is equivalent to:
       // Iterator* iter = table->NewIterator();
       // iter->Seek("");
       // return iter;
   // If you are going to immediately seek to another place in the
   // returned iterator, it will be faster to use NewIterator()
   // and avoid the extra seek.
   Iterator* GetIterator() const;
   ```

3. 每个函数定义时要以注释说明函数功能和实现要点，如使用的漂亮代码、实现的简要步骤、如此实现的理由、为什么前半部分要加锁而后半部分不需要。



## 变量注释

1. 每个类数据成员都应该添加注释，，如果变量可以接受NULL 或-1等警戒值（sentinel values），须说明之

```c++
private:
// Keeps track of the total number of entries in the table.
// Used to ensure we do not go over the limit. -1 means
// that we don't yet know how many entries the table has.
int num_total_entries_;
```

2. 全局变量和数据成员相似，所有全局变量（常量）也应注释说明含义及用

## 实现注释

1. 对于实现代码中巧妙的、晦涩的、有趣的、重要的地方加以注释
2. 代码前注释，出彩的或复杂的代码块前要加注释

3. 比较隐晦的地方要在行尾加入注释，可以在代码之后**空两格**加行尾注释

   ```c++
   // If we have enough memory, mmap the data portion too.
   mmap_budget = max<int64>(0, mmap_budget - index_->length());
   if (mmap_budget >= data_size_ && !MmapData(mmap_chunk_bytes, mlock))
   return; // Error already logged.
   ```

4. 前后相邻几行都有注释，可以适当调整使之可读性更好：

   ```c++
   ...
   DoSomething(); // Comment here so the comments line up.
   DoSomethingElseThatIsLonger(); // Comment here so there are two spaces between
   // the code and the comment.
   ...
   ```

   

5. 向函数传入、布尔值或整数时,要注释说明

   ```c++
   bool success = CalculateSomething(interesting_value,
   			10,
   			false,
   			NULL); // What are these arguments??
   
   bool success = CalculateSomething(interesting_value,
   			10, // Default base value.
   			false, // Not the first time we're calling
   			this.
   			NULL); // No callback.
   
   // 使用常量或描述性变量：
   const int kDefaultBaseValue = 10;
   const bool kFirstTimeCalling = false;
   Callback *null_callback = NULL;
   bool success = CalculateSomething(interesting_value,
   			kDefaultBaseValue,
   			kFirstTimeCalling,
   			null_callback)
   ```

   

6. 注意永远不要用自然语言翻译代码作为注释



