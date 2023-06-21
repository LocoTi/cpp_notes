# Effective Modern C++42

[Effective Modern C++42招独家技巧助你改善C++11和C++14的高效用法笔记_网络资源是无限的-CSDN博客](https://blog.csdn.net/fengbingchun/article/details/104136592)

注：(1). 以下测试代码既可以在 Windows 下执行也可以在 Linux 执行。(2). 个人感觉中文版有些内容不如直接看英文版理解的更透彻，因此下面有些中文也同时给出了对应的英文。

C++11 被最广泛接受的特性可能莫过于移动语义，而移动语义的基础在于区分左值表达式和右值表达式。因为，一个对象是右值意味着能够对其实施移动语义，而左值则一般不然。从概念上说 (实践上并不总是成立)，右值对应的是函数返回的临时对象，而左值对应的是可指涉的对象，而指涉的途径则无论通过名字、指针，还是左值引用皆可。

有一种甄别表达式是否是左值的实用方法富有启发性，那就是检查能否取得该表达式的地址。如果可以取得，那么该表达式基本上可以判定是左值。如果不可取得，则其通常是右值。这种方法之所以说富有启发性，是因为它让你记得，表达式的型别 (type) 与它是左值还是右值没有关系。换言之，给定一型别 T，则既有 T 型别的左值，也有 T 型别的右值。

在函数调用中，调用方的表达式，称为函数的实参。实参的用处，是初始化函数的形参。实参和形参有着重大的区别，因为形参都是左值，而用来作为其初始化依据的实参，则既可能是右值，也可能是左值。



## **1.** **理解模板型别推导 (Understand template type deduction)**



```
//template<typename T>
//void f(ParamType param);
 
template<typename T>
void f(T& param) {} // param是个引用
 
template<typename T>
void f2(T* param) {} // param现在是个指针
 
template<typename T>
void f3(T&& param) {} // param现在是个万能引用
 
template<typename T>
void f4(T param) {} // param现在是按值传递
 
// 以编译期常量形式返回数组尺寸(该数组形参未起名字，因为我们只关系其含有的元素个数)
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept // 将该函数声明为constexpr，能够使得其返回值在编译期就可用。从而就可以在
{						   // 声明一个数组时，指定其尺寸和另一数组相同，而后者的尺寸则从花括号初始化式(braced initializer)计算得出
	return N;
}
 
void someFunc(int, double) {} // someFunc是个函数，其型别为void(int, double)
 
int test_item_1()
{
	//f(expr); // 已某表达式调用f
	// 在编译期，编译器会通过expr推导两个型别：一个是T的型别，另一个是ParamType的型别，这两个型别往往不一样
 
	int x = 27; // x的型别是int
	const int cx = x; // cx的型别是const int
	const int& rx = x; // rx是x的型别为const int的引用
 
	f(x); // T的型别是int, param的型别是int&
	f(cx); // T的型别是const int, param的型别是const int&
	f(rx); // T的型别是const int, param的型别是const int&, 注意：即使rx具有引用型别，T也并未被推导成一个引用，原因在于，rx的引用性(reference-ness)会在型别推导过程中被忽略
 
	const int* px = &x; // px is ptr to x as a const int
	f2(&x); // T is int, param's type is int*
	f2(px); // T is const int, param's type is const int*
 
	f3(x); // x is lvalue, so T is int&, param's type is also int&
	f3(cx); // cx is lvalue, so T is const int&, param's type is also const int&
	f3(rx); // rx is lvalue, so T is const int&, param's type is also const int&
	f3(27); // 27 is rvalue, so T is int, param's type is therefore int&&
 
	// param是个完全独立于cx和rx存在的对象----是cx和rx的一个副本
	f4(x); // T's and param's types are both int
	f4(cx); // T's and param's types are again both int
	f4(rx); // T's and param's types are still both int
 
	const char* const ptr = "Fun with pointers"; // ptr is const pointer to const object
	f4(ptr); // pass arg of type const char* const
 
	const char name[] = "J. P. Briggs"; // name's type is const char[13]
	const char* ptrToName = name; // array decays to pointer
 
	f4(name); // name is array, but T deduced as const char*
	f(name); // pass array to f, T的型别推导结果是const char[13], 而f的形参(该数组的一个引用)型别则被推导为const char (&)[13]
 
	int keyVals[] = {1, 3, 7, 9, 11, 22, 35};
	fprintf(stdout, "array length: %d\n", arraySize(keyVals)); // 7
	int mappedVals[arraySize(keyVals)]; // mappedVals被指定与之相同
	std::array<int, arraySize(keyVals)> mappedVals2; // mappedVals2也指定为7个元素
 
	f4(someFunc); // param被推导为函数指针(ptr-to-func)，具体型别是void (*)(int, double)
	f(someFunc); // param被推导为函数引用(ref-to-func), 具体型别是void (&)(int, double)
 
	return 0;
}
```



T 的型别推导结果，不仅仅依赖 expr 的型别，还依赖 ParamType 的形式。具体要分三种情况讨论：



(1).ParamType 具有指针或引用型别，但不是万能引用 (universal reference)：若 expr 具有引用型别，先将引用部分忽略；然后对 expr 的型别和 ParamType 的型别执行模式匹配，来决定 T 的型别。



(2).ParamType 是一个万能引用 (universal reference)：此类形参的声明方式类似右值引用(即在函数模板中持有型别形参 T 时，万能引用的声明型别写作 T&&)，但是当传入的实参是左值时，其表现会有所不同。如果 expr 是个左值，T 和 ParamType 都会被推导为左值引用。这个结果具有双重的奇特之处：首先，这是在模板型别推导中，T 被推导为引用型别的唯一情形。其次，尽管在声明时使用的是右值引用语法，它的型别推导结果却是左值引用。如果 expr 是个右值，则应用” 常规”(即情况 1 中的)规则。当遇到万能引用时，型别推导规则会区分实参是左值还是右值。而非万能引用是从来不会做这样的区分的。



(3).ParamType 既非指针也非引用：当 ParamType 既非指针也非引用时，我们面对的就是所谓按值传递了。一如之前，若 expr 具有引用型别，则忽略其引用部分。忽略 expr 的引用性之后，若 expr 是个 const 对象，也忽略之。若其是个 volatile 对象，同样忽略之 (volatile 对象不常用，它们一般仅用于实现设备驱动程序)。



数组实参：数组型别 (array type) 有别于指针型别，尽管有时它们看起来可以互换。形成这种假象的主要原因是，在很多语境下，数组会退化成指涉到其首元素的指针。**可以利用声明数组引用这一能力创造出一个模板，用来推导出数组含有的元素个数**。



函数实参：数组并非 C++ 中唯一可以退化为指针之物。函数型别也同样会退化成函数指针，并且我们针对数组型别推导的一切讨论都适用于函数及其向函数指针的退化。



**要点速记：(1).** **在模板型别推导过程中，具有引用型别的实参会被当成非引用型别来处理。换言之，其引用性会被忽略。(2).** **对万能引用形态进行推导时，左值实参会进行特殊处理。(3).** **对按值传递的形参进行推导时，若实参型别中带有 const** **或 volatile** **饰词，则它们还是会被当作不带 const** **或 volatile** **饰词的型别来处理。(4).** **在模板型别推导过程中，数组或函数型别的实参会退化成对应的指针 (arguments that are array or function names decay to pointers)****，除非它们被用来初始化引用**。



## **2.** **理解 auto** **型别推导 (Understand auto type deduction)**



```
//template<typename T>
//void f(ParamType param);
 
void someFunc2(int, double) {} // someFunc是个函数，其型别为void(int, double)
 
/*auto createInitlist()
{
	return {1, 2, 3}; // error: can't deduce type for {1, 2, 3}
}*/
 
int test_item_2()
{
	//f(expr); // 已某表达式调用f
	// 当某变量采用auto来声明时，auto就扮演了模板中的T这个角色，而变量的型别饰词则扮演的是ParamType的角色
	auto x = 27; // x的型别饰词(type specifier)就是auto自身, x既非指针也非引用
	const auto cx = x; // 型别饰词成了const auto, cx既非指针也非引用
	const auto& rx = x; // 型别饰词又成了const auto&, rx是个引用，但不是万能引用
 
	auto&& uref1 = x; // x的型别是int,且是左值，所以uref1的型别是int&
	auto&& uref2 = cx; // cx的型别是const int, 且是左值，所以uref2的型别是const int&
	auto&& uref3 = 27; // 27的型别是int,且是右值，所以uref3的型别是int&&
 
	const char name[] = "R. N. Briggs"; // name的型别是const char[13]
	auto arr1 = name; // arr1's type is const char*
	auto& arr2 = name; // arr2's type is const char (&)[13]
 
	auto func1 = someFunc2; // func1's type is void(*)(int, double)
	auto& func2 = someFunc2; // func2's type is void(&)(int, double)
 
	// 若要声明一个int，并将其初始化为值27，C++98中有两种可选语法
	int x1 = 27;
	int x2(27);
	// 而C++11为了支持统一初始化(uniform initialization),增加了下面的语法选项
	int x3 = {27};
	int x4{27};
 
	auto x1_1 = 27; // type is int, value is 27
	auto x2_1(27); // type is int, value is 27
	auto x3_1 = {27}; // type is std::initializer_list<int>, value is {27}
	auto x4_1{27}; // type is std::initializer_list<int>, value is {27}
	//auto x5_1 = {1, 2, 3.0}; // error, can't deduce T for std::initializer_list<T> 
 
	std::vector<int> v;
	auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14
	//resetV({1, 2, 3}); // error, can't deduce type for {1, 2, 3}
 
	return 0;
}
```



除了一个奇妙的例外情况以外，auto 型别推导就是模板型别推导。在采用 auto 进行变量声明中，型别饰词取代了 ParamType，所以也存在三种情况：(1). 型别饰词是指针或引用，但不是万能引用 (universal reference)。(2). 型别饰词是万能引用。(3). 型别饰词既非指针也非引用。



**当用于 auto** **声明变量的初始化表达式是使用大括号括起时，推导所得的型别就属于 std::initializer_list**。这么一来，如果型别推导失败 (例如，大括号里的值型别不一)，则代码就通不过编译。对于大括号初始化表达式的处理方式是 auto 型别推导和模板型别推导的唯一不同之处。当采用 auto 声明的变量使用大括号初始化表达式进行初始化时，推导所得的型别是 std::initializer_list 的一个实例型别，但模板型别却不会。



C++14 允许使用 auto 来说明函数返回值需要推导，而且 C++14 中的 lambda 式也会在形参声明中用到 auto。然而，这些 auto 用法是在使用模板型别推导而非 auto 型别推导。所以，带有 auto 返回值的函数若要返回一个大括号括起来的初始化表达式，是通不过编译的。同样地，用 auto 来指定 C++14 中 lambda 式的形参型别时，也不能使用大括号括起的初始化表达式。



**要点速记：(1).** **在一般情况下，auto** **型别推导和模板型别推导是一模一样的，但是 auto** **型别推导会假定用大括号括起的初始化表达式代表一个 std::initializer_list****，但模板型别推导却不会。(2).** **在函数返回值或 lambda** **式的形参中使用 auto****，意思是使用模板型别推导而非 auto** **型别推导**。



auto 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/51834927



## **3.** **理解 decltype(Understand decltype)** **模板**



```
class Widget3 {};
bool f5(const Widget3& w) { return true; } // decltype(w) is const Widget3&; decltype(f5) is bool(const Widgeet3&)
 
template<typename Container, typename Index>
// 这里的auto只为说明这里使用了C++11中的返回值型别尾序语法(trailing return type syntax),即该函数的返回值型别将在形参列表之后(在"->"之后)
// 尾序返回值的好处在于，在指定返回值型别时可以使用函数形参
//auto authAndAccess(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i]) // C++11
decltype(auto) authAndAccess(Container&& c, Index i) // C++14, c is now a universal reference
{
	return std::forward<Container>(c)[i];
}
 
struct Point {
	int x, y; // decltype(Point::x) is int; decltype(Point::y) is int
};
 
decltype(auto) ff3_1()
{
	int x = 0;
	return x; // decltype(x)是int,所以ff3_1返回的是int
	//return (x); // decltype((x))是int&,所以ff3_1返回的是int&
}
 
int test_item_3()
{
	const int i = 0; // decltype(i) is const int
	Widget3 w; // decltype(w) is Widget3
	if (f5(w)) {} // decltype(f5(w)) is bool
	std::vector<int> v; // decltype(v) is vector<int>
 
	const Widget3& cw = w;
	auto myWidget31 = cw; // auto型别推导:myWidget31的型别是Widget3
	decltype(auto) myWidget32 = cw; // decltype型别推导:myWidget32的型别是const Widget3&
 
	return 0;
}
```



对于给定的名字或表达式，decltype 能告诉你该名字或表达式的型别。与模板和 auto 的型别推导过程相反，decltype 一般只会鹦鹉学舌，返回给定的名字或表达式的确切型别而已。



C++11 中，decltype 的主要用途大概就在于声明那些返回值型别依赖于形参型别的函数模板。



C++11 允许对单表达式的 lambda 式的返回值型别实施推导，而 C++14 则将这个允许范围扩张到了一切 lambda 式和一切函数，包括那些多表达式。



C++14 中的 decltype(auto) 并不限于在函数返回值型别处使用。在变量声明的场合上，若你也想在初始化表达式处应用 decltype 型别推导规则，也可以照样便宜行事。



容器的传递方式是对非常量的左值引用 (lvalue-reference-to-non-const)。



**要点速记：(1).** **绝大多数情况下，decltype** **会得出变量或表达式的型别而不作任何修改。(2).** **对于型别为 T** **的左值表达式，除非该表达式仅有一个名字，否则 decltype** **总是得出型别 T&(For lvalue expressions of type T other than names, decltype always reports a type of T&)****。(3).C++14** **支持 decltype(auto)****，和 auto** **一样，它会从其初始化表达式出发来推导型别，但是它的型别推导使用的是 decltype** **的规则**。



decltype 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52504519



## **4.** **掌握查看型别推导结果的方法 (Know how to view deduced types)**



```
int test_item_4()
{
	const int theAnswer = 42;
	auto x = theAnswer; // int
	auto y = &theAnswer; // const int*
	fprintf(stdout, "%s, %s\n", typeid(x).name(), typeid(y).name());
 
	return 0;
}
```



IDE 编辑器：IDE 中的代码编辑器通常会在你将鼠标指针悬停至某个程序实体，如变量、形参、函数等时，显示出该实体的型别。IDE 显示的型别信息不可靠。



编译器诊断信息：想要让编译器显示其推导出的型别，一条有效的途径是使用该型别导致某些编译错误。而报告错误的消息几乎肯定会提及导致该错误的型别。



运行时输出：使用 printf 来显示型别信息，这种方法只有到了运行期才能使用，却可以对于型别输出的格式提供完全的控制。std::type_info::name 并不可靠。



**要点速记：(1).** **利用 IDE** **编辑器、编译器错误信息和 Boost.TypeIndex** **库常常能够查看到推导而得的型别。(2).** **有些工具产生的结果可能会无用，或者不准确。所以理解 C++** **型别推导规则是必要的**。



## **5.** **优先选用 auto****，而非显式型别声明 (Prefer auto to explicit type declarations)**



```
class Widget5 {};
bool operator<(const Widget5& lhs, const Widget5& rhs) { return true; }
 
int test_item_5()
{
	int x1; // potentially uninitialized
	//auto x2; // error, initializer required
	auto x3 = 0; // fine, x's value is well-defined
 
	auto derefUPLess = [](const std::unique_ptr<Widget5>& p1, const std::unique_ptr<Widget5>& p2) { return *p1 < *p2; }; // comparison func. for Widget5 pointed to by std::unique_ptrs
	auto derefLess = [](const auto& p1, const auto& p2) { return *p1 < *p2; }; // C++14 comparison function for values pointed to by anything pointer-like
 
	// bool(const std::unique_ptr<Widget5>&, const std::unique_ptr<Widget5>&) // C++11 signature for std::unique_ptr<Widget5> comparison function
	std::function<bool(const std::unique_ptr<Widget5>&, const std::unique_ptr<Widget5>&)> func;
 
	// 在C++11中，不用auto也可以声明derefUPLess
	std::function<bool(const std::unique_ptr<Widget5>&, const std::unique_ptr<Widget5>&)> derefUPLess2 = [](const std::unique_ptr<Widget5>& p1, const std::unique_ptr<Widget5>& p2) { return *p1 < *p2; };
 
	std::vector<int> v{1, 2, 3};
	unsigned sz1 = v.size(); // 不推荐,32位和64位windows上,unsigned均是32位，而在64位windows上，std::vector<int>::size_type则是64位
	auto sz2 = v.size(); // 推荐，sz2's type is std::vector<int>::size_type
	
	std::unordered_map<std::string, int> m;
	for (const std::pair<std::string, int>& p : m) {} // 显式型别声明，不推荐,the key part of a std::unordered_map is const， so the type of std::pair in the hash table is std::<const std::string, int>, 需要进行隐式转换，会产生临时对象
	for (const auto& p : m) {} // 推荐
 
	return 0;
}
```



用 auto 声明的变量必须初始化。



在 C++14 中，lambda 表达式的形参都可以使用 auto。



std::function 是 C++11 标准库中的一个模板。函数指针只能指涉 (point) 到函数，而 std::function 却可以指涉 (refer to) 任何可调用对象，即任何可以像函数一样实施调用之物。正如你若要创建一个函数指针就必须指定欲指涉到的函数的型别(即该指针指涉到的函数的签名)，你若要创建一个 std::function 对象就必须指定欲指涉的函数的型别。



使用 std::function 和使用 auto 有所不同：使用 auto 声明的、存储着一个闭包 (closure) 的变量和该闭包是同一型别，从而它要求的内存量也和该闭包一样。而使用 std::function 声明的、存储着一个闭包的变量是 std::function 的一个实例，所以不管给定的签名 (signature) 如何，它都占有固定尺寸的内存，而这个尺寸对于其存储的闭包而言并不一定够用。如果是这样的话，std::function 的构造函数就会分配堆上的内存来存储该闭包。从结果上看，std::function 对象一般都会比使用 auto 声明的变量使用更多内存。通过 std::function 来调用闭包几乎必然会比通过使用 auto 声明的变量来调用同一闭包要来得慢。



std::function 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52562918



auto 也并不完美，每个 auto 变量的型别都是从它的初始化表达式推导出来的，而有些初始化表达式的型别既不符合期望也不符合要求。



显式的写出型别经常是画蛇添足，带来各种微妙的偏差，有些关乎正确性，有些关乎效率，或是两者都受影响。还有，auto 型别可以随着其初始化表达式的型别变化而自动随之改变。



**要点速记：(1).auto** **变量必须初始化，基本上对会导致兼容性和效率问题的型别不匹配现象免疫，还可以简化重构流程，通常也比显式指定型别要少打一些字。(2).auto** **型别的变量都有着条款 2** **和条款 6** **中所描述的毛病**。



auto 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/51834927



## **6.** **当 auto** **推导的型别不符合要求时，使用带显式型别的初始化物习惯用法 (Use the explicitly typed initializer idiom when auto deduces undesired types)**



```
class Widget6 {};
std::vector<bool> features(const Widget6& w)
{
	return std::vector<bool>{true, true, false, false, true, false};
}
 
void processWidget6(const Widget6& w, bool highPriority) {}
 
double calcEpsilon() { return 1.0; }
 
int test_item_6()
{
	Widget6 w;
	bool highPriority =features(w)[5]; // 正确,显式声明highPriority的型别
	processWidget6(w, highPriority);
 
	// 把highPriority从显示型别改成auto
	auto highPriority2 = features(w)[5]; // highPriority2的型别由推导而得,std::vector<bool>的operator[]的返回值并不是容器中一个元素的引用(单单bool是个例外),返回的是个std::vector<bool>::reference型别的对象,返回一个std::vector<bool>型别的临时对象
	processWidget6(w, highPriority2); // undefined behavior, highPriority2 contains dangling pointer(空悬指针)
 
	auto highPriority3 = static_cast<bool>(features(w)[5]); // 正确
	processWidget6(w, highPriority3);
 
	float ep = calcEpsilon(); // 隐式转换 double-->float,这种写法难以表明"我故意降低了函数的返回值精度"
	auto ep2 = static_cast<float>(calcEpsilon()); // 推荐
 
	return 0;
}
```



std::vector<bool> 是 vector 的特殊版本，用于 bool 类型的元素并优化空间，存储每个值仅占用一个位而不是一个字节 (each value is stored in a single bit)。



std::vector<bool>::reference 是个代理类的实例。所谓代理类，就是指为了模拟或增广其它型别的类 (a class that exists for the purpose of emulating and augmenting the behavior of some other type)。一个普遍的规律是，” 隐形”代理类和 auto 无法和平共处。问题在于 auto 没有推导成为你想推导出来的型别。解决方案应该是强制进行另一次型别转换，这种方法称为带显式型别的初始化物习惯用法。



带显式型别的初始化物习惯用法要求使用 auto 声明变量，但针对初始化表达式进行强制型别转换，转换成你想要 auto 推导出来的型别。



**要点速记：(1).”** **隐形”** **的代理型别可以导致 auto** **根据初始化表达式推导出”** **错误的”** **型别。(2).** **带显式型别的初始化物习惯用法强制 auto** **推导出你想要的型别**。



## **7.** **在创建对象时注意区分 ()** **和 {}(Distinguish between()and{}when creating objects)**



```
class Widget7 {
public:
	Widget7(int i, bool b) {} // constructor not declaring std::initializer_list params
	Widget7(int i, double d) {}
	Widget7(std::initializer_list<long double> il) { fprintf(stdout, "std::initializer_list params\n"); }
	Widget7() = default;
	Widget7(int) {}
 
	operator float() const { return 1.0f; } // 强制转换成float型别, 注意：此函数的作用,下面的w13和w15
 
private:
	int x{0}; // fine, x's default value is 0
	int y = 0; // also fine
	//int z(0); // error
};
 
int test_item_7()
{
	int x(0); // 初始化物在小括号内
	int y = 0; // 初始化物在等号之后
	int z{0}; // 初始化物在大括号内
	int z2 = {0}; // 使用等号和大括号来指定初始化物，一般C++会把它和只有大括号的语法同样处理
 
	Widget7 w1; // call default constructor
	Widget7 w2 = w1; // not an assignment; calls copy constructor
	w1 = w2; // an assignment; calls copy operator=
 
	std::vector<int> v{1, 3, 5}; // v's initial content is 1, 3, 5
 
	std::atomic<int> ai1{0}; // fine
	std::atomic<int> ai2(0); // fine
	//std::atomic<int> ai3 = 0; // error
 
	double a{std::numeric_limits<double>::max()}, b{std::numeric_limits<double>::max()}, c{std::numeric_limits<double>::max()};
	//int sum1{a + b + c}; // error, sum of doubles may not be expressible as int
	int sum2(a + b + c); // okey(value of expression truncated to an int)
	int sum3 = a + b + c; // okey(value of expression truncated to an int)
 
	Widget7 w3(10); // call Widget7 constructor with argument 10
	Widget7 w4(); // most vexing parse! declares a function named w4 that returns a Widget7
	Widget7 w5{}; // call Widget7 constructor with no args
 
	Widget7 w6(10, true); // calls first constructor
	Widget7 w7{10, true}; // alse calls first constructor, 假设没有Widget7(std::initializer_list<long double>)构造函数
	Widget7 w8(10, 5.0); // calls second constructor
	Widget7 w9{10, 5.0}; // also calls second constructor, 假设没有Widget7(std::initializer_list<long double>)构造函数
 
	Widget7 w10{10, true}; // 使用大括号，调用的是带有std::initializer_list型别形参的构造函数(10和true被强制转换为long double)
	Widget7 w11{10, 5.0}; // 使用大括号，调用的是带有std::initializer_list型别形参的构造函数(10和5.0被强制转换为long double) 
 
	Widget7 w12(w11); // 使用小括号，调用的是拷贝构造函数
	Widget7 w13{w11}; // 使用大括号，调用的是带有std::initializer_list型别形参的构造函数(w11的返回值被强制转换成float,随后float又被强制转换成long double)
	Widget7 w14(std::move(w11)); // 使用小括号，调用的是移动构造函数
	Widget7 w15{std::move(w11)}; // 使用大括号，调用的是带有std::initializer_list型别形参的构造函数(和w13的结果理由相同)
	
	Widget7 w16{}; // call Widget7 constructor with no args,调用无参的构造函数，而非调用带有std::initializer_list型别形参的构造函数
	Widget7 w17({}); // 调用带有std::initializer_list型别形参的构造函数，传入一个空的std::initializer_list
	Widget7 w18{{}}; // 调用带有std::initializer_list型别形参的构造函数，传入一个空的std::initializer_list
 
	std::vector<int> v1(10, 20); // 调用了形参中没有任何一个具备std::initializer_list型别的构造函数，结果是：创建了一个含有10个元素的std::vector，所有的元素的值都是20
	std::vector<int> v2{10, 20}; // 调用了形参中含有std::initializer_list型别的构造函数，结果是：创建了一个含有2个元素的std::vector，元素的值分别为10和20
	fprintf(stdout, "v1 length: %d, v2 length: %d\n", v1.size(), v2.size());
 
	return 0;
}
```



指定初始化值的方式包括使用小括号、使用等号，或是使用大括号。



C++11 引入了统一初始化 (uniform initialization)：单一的、至少从概念上可以用于一切场合、表达一切意思的初始化。它的基础是大括号形式或称为大括号初始化 (braced initialization)。



大括号同样可以用来为非静态成员指定默认初始化值，这项能力 (在 C++11 中新加入的能力) 也可以使用”=”的初始化语法，却不能使用小括号。



不可复制的对象 (如 std::atomic 型别的对象) 可以采用大括号和小括号来进行初始化，却不能使用”=”。



大括号初始化有一项新特性，就是它禁止内建型别之间进行隐式窄化型别转换 (narrowing conversion)。如果大括号内的表达式无法保证能够采用进行初始化的对象来表达，则代码不能通过编译。而采用小括号和”=” 的初始化则不会进行窄化型别转换检查。



大括号初始化的另一项值得一提的特征是，它对于 C++ 的最令人苦恼之解析语法 (most vexing parse) 免疫。C++ 规定：任何能够解析为声明的都要解析为声明，而这会带来副作用。所谓最令人苦恼之解析语法就是说，程序员本来想要以默认方式构造一个对象，结果却一不小心声明了一个函数。这个错误的根本原因在于构造函数调用语法。



**大括号初始化的缺陷在于伴随它有时会出现的意外行为**：这种行为源于大括号初始化物、std::initializer_list 以及构造函数重载决议之间的纠结关系。这几者之间的相互作用可以使得代码看起来是要做某一件事，但实际上是在做另一件事。如果使用大括号初始化物来初始化一个使用 auto 声明的变量，那么推导出来的型别就会成为 std::initializer_list，尽管用其它方式使用相同的初始化物来声明变量就能够得出更符合直觉的型别。在构造函数被调用时，只要形参中没有任何一个具备 std::initializer_list 型别，那么小括号和大括号的意义就没有区别。**如果，有一个或多个构造函数声明了任何一个具备** **std::initializer_list** **型别的形参，那么采用了大括号初始化语法的调用语句会强烈地优先选用带有 std::initializer_list** **型别形参的重载版本**。即使是平常会执行复制 (copy) 或移动的构造函数也可能被带有 std::initializer_list 型别形参的构造函数劫持。只有在找不到任何办法把大括号初始化物中的实参转换成 std::initializer_list 模板中的型别时，编译器才会退而去检查普通的重载决议 (normal overload resolution)。空大括号对表示的是” 没有实参”，而非”空的 std::initializer_list”。



**要点速记：(1).** **大括号初始化可以应用的语境最为宽泛，可以阻止隐式窄化型别转换，还对最令人苦恼之解析语法免疫。(2).** **在构造函数重载决议期间，只要有任何可能，大括号初始化物就会与带有 std::initializer_list** **型别的形参相匹配，即使其它重载版本有着貌似更加匹配的形参表。(3).** **使用小括号还是大括号，会造成结果大相径庭的一个例子是：使用两个实参来创建一个 std::vector<** **数值型别 >** **对象。(4).** **在模板内容进行对象创建时，到底应该使用小括号还是大括号会成为一个棘手问题**。



## **8.** **优先选用 nullptr****，而非 0** **或 NULL(Prefer nullptr to 0 and NULL)**



```
void f8(int) { fprintf(stdout, "f8(int)\n"); }
void f8(bool) { fprintf(stdout, "f8(bool)\n"); }
void f8(void*) { fprintf(stdout, "f8(void*)\n"); }
 
class Widget8 {};
int f8_1(std::shared_ptr<Widget8> spw) { return 0; }
double f8_2(std::unique_ptr<Widget8> upw) { return 1.f; }
bool f8_3(Widget8* pw) { return false; }
 
template<typename FuncType, typename MuxType, typename PtrType>
//auto lockAddCall(FuncType func, MuxType& mutex, PtrType ptr) -> decltype(func(ptr)) // C++11
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr) // C++14
{
	using MuxGuard = std::lock_guard<std::mutex>; // C++11 typedef
	MuxGuard g(mutex);
	return func(ptr);
}
 
int test_item_8()
{
	f8(0); // calls f8(int), not f8(void*)
	//f8(NULL); // might not compile, but typically calls f8(int), never calls f8(void*)
	f8(nullptr); // calls f(void*) overload
 
	std::mutex f1m, f2m, f3m;
	//auto result1 = lockAndCall(f8_1, f1m, 0); // error, ‘void result1’ has incomplete type
	//auto result2 = lockAndCall(f8_2, f2m, NULL); // error: ‘void result2’ has incomplete type
	auto result3 = lockAndCall(f8_3, f3m, nullptr);
 
	return 0;
}
```



字面常量 0 的型别是 0，而非指针。当 C++ 在只能使用指针的语境中发现了一个 0，它也会把它勉强解释为空指针，但说到底这是一个不得已而为之的行为。C++ 的基本观点还是 0 的型别是 int，而非指针。



nullptr 的优点在于，它不具备整型型别。实话实说，它也不具备指针型别 (pointer type)。nullptr 的实际型别是 std::nullptr_t，std::nullptr_t 的定义被指定为 nullptr 的型别。型别 std::nullptr_t 可以隐式转换到所有的裸指针型别 (raw pointer type)，这就是为何 nullptr 可以扮演所有型别指针的原因。



**要点速记：(1).** **相对于 0** **或 NULL****，优先选用 nullptr****。(2).** **避免在整型和指针型别之间重载。**



nullptr 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/51793497



## **9.** **优先选用别名声明，而非 typedef(Prefer alias declarations to typedefs)**



```
class Widget9 {};
 
typedef void (*FP1)(int, const std::string&);
using FP2 = void (*)(int, const std::string&);
 
template<typename T>
using MyAllocList1 = std::list<T/*, MyAlloc<T>*/>; // C++11,  MyAllocList1<T>是std::list<T, MyAlloc<T>>的同义词
 
template<typename T>
struct MyAllocList2 { // MyAllocList<T>::type 是std::list<T, MyAlloc<T>>的同义词
	typedef std::list<T/*, MyAlloc<T>*/> type;
};
 
template<typename T>
class Widget9_2 { // Widget9_2<T>含一个MyAllocList2<T>型别的数据成员
private:
	typename MyAllocList2<T>::type list; // MyAllocList2<T>::type代表一个依赖于模板型别形参(T)的型别，所以MyAllocList2<T>::type称为带依赖型别，C++中规则之一就是带依赖型别必须前面加个typename
};
 
template<typename T>
class Widget9_1 {
private:
	MyAllocList1<T> list; // 不再有"typename"和"::type"
};
 
int test_item_9()
{
	typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS1;
	using UPtrMapSS2 = std::unique_ptr<std::unordered_map<std::string, std::string>>; // C++11, alias declaration
 
	MyAllocList1<Widget9> lw1;
	MyAllocList2<Widget9>::type lw2;
 
	typedef const char cc;
	std::remove_const<cc>::type a; // char a
	std::remove_const<const char*>::type b; // const char* b
 
	typedef int&& rval_int;
	typedef std::remove_reference<int>::type A;
 
	//std::remove_const<T>::type // C++11: const T --> T
	//std::remove_const_t<T>     // C++14中的等价物
	//template<class T>
	//using remove_const_t = typename remove_const<T>::type;
 
	//std::remove_reference<T>::type // C++11: T&/T&& --> T
	//std::remove_reference_t<T>     // C++14中的等价物
	//template<class T>
	//using remove_reference_t = typename remove_reference<T>::type;
 
	return 0;
}
```



别名声明可以模板化 (这种情况下它们被称为别名模板，alias template)，typedef 就不行。



C++11 以型别特征 (type trait) 的形式给了程序员以执行此类变换的工具。型别特征是在头文件 < type_traits > 给出的一整套模板。该头文件中有几十个型别特征，它们并非都是执行型别变换功能的用途，但其中派此用途的部分则提供了可预测的接口。



每个 C++11 中的变换 std::transformation<T>::type，都有一个 C++14 中名为 std::transformation_t 的对应别名模板。



**要点速记：(1).typedef** **不支持模板化，但别名声明支持。(2).** **别名模板可以让人免写”::type”** **后缀，并且在模板内，对于内嵌 typedef** **的引用经常要求加上 typename** **前缀**。



别名声明更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/81259210



## **10.** **优先选用限定作用域的枚举型别，而非不限作用域的枚举型别 (Prefer scoped enums to unscoped enums)**



```
std::vector<std::size_t> primeFactors(std::size_t x) { return std::vector<std::size_t>(); }
 
enum class Status; // 前置声明, 默认底层型别(underlying type)是int
enum class Status2: std::uint32_t; // Status2的底层型别是std::uint32_t
enum Color: std::uint8_t; // 不限范围的枚举型别的前置声明，底层型别是std::uint8_t
 
int test_item_10()
{
	enum Color1 { black, white, red }; // 不限范围的(unscoped)枚举型别：black, white, red所在作用域和Color1相同
	//auto white = false; // error, white already declared in this scope
	Color1 c1 = black;
 
	enum class Color2 { black2, white2, red2 }; // C++11, 限定作用域的(scoped)枚举型别:black2, white2, red2所在作用域被限定在Color2内
	auto white2 = false; // 没问题，范围内并无其它"white2"
	//Color2 c1 = black2; // 错误，范围内并无名为"black2"的枚举量
	Color2 c2 = Color2::black2; // fine
	auto c3 = Color2::black2; // also fine
 
	if (c1 < 14.5) // 将Color1型别和double型别值作比较,怪胎
		auto factors = primeFactors(c1);
 
	//if (c2 < 14.5) // 错误，不能将Color型别和double型别值作比较
	//	auto facotrs = primeFactors(c2); // 错误，不能将Color2型别传入要求std::size_t型别形参的函数
 
	return 0;
}
```



一个通用规则，如果在一对大括号里声明一个名字，则该名字的可见性就被限定在括号括起来的作用域内。但这个规则不适用于 C++98 风格的枚举型别中定义的枚举量。这些枚举量的名字属于包含着这个枚举型别的作用域，这就意味着在此作用域内不能有其它实体取相同的名字。



由于限定作用域的枚举型别是通过”enum class” 声明的，所有有时它们也被称为枚举类。



限定作用域的枚举型别带来的名字空间污染降低，已经是” 应该优先选择它，而不是不限范围的枚举型别” 的足够理由。但是限定作用域的枚举型别还有第二个优势：它的枚举量是更强型别的 (strongly typed)。不限范围的枚举型别中的枚举量可以隐式转换到整数型别 (并能够从此处进一步转换到浮点型别)。**从限定作用域的枚举型别到任何其它型别都不存在隐式转换路径**。限定作用域的枚举型别可以进行前置声明，C++11 中，不限范围的枚举型别也可以进行前置声明，但须得在完成一些额外工作之后。这些额外工作是由以下事实带来的：一切枚举型别在 C++ 里都会由编译器来选择一个整数型别作为其底层型别。



为了节约使用内存，编译器通常会为枚举型别选用足够表示枚举量取值的最小底层型别。在某些情况下，编译器会用空间来换取时间，而在这样的情况下，它们可能会不选择只具备最小可容尺寸的型别，但是它们当然需要具备优化空间的能力。为了使这种设计成为可能，C++98 就只提供了枚举型别定义 (即列出所有枚举量) 的支持，枚举型别声明则不允许。



限定作用域的枚举型别的底层型别 (underlying type) 是已知的，默认地是 int；而对于不限范围的枚举型别，你可以指定这个底层型别。如果要指定不限范围的枚举型别的底层型别，做法和限定作用域的枚举型别一样。这样作了以后，不限范围的枚举型别也能够进行前置声明了。底层型别指定同样也可以在枚举型别定义中进行。



**要点速记：(1).C++98** **风格的枚举型别，现在称为不限范围的枚举型别。(2).** **限定作用域的枚举型别仅在枚举型别内可见。它们只能通过强制型别转换以转换至其它型别。(3).** **限定作用域的枚举型别和不限范围的枚举型别都支持底层型别指定。限定作用域的枚举型别的默认底层型别是 int****，而不限范围的枚举型别没有默认底层型别。(4).** **限定作用域的枚举型别总是可以进行前置声明，而不限范围的枚举型别却只有在指定了默认底层型别的前提下才可以进行前置声明**。



enum class 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/78535754



## **11.** **优先选用删除函数，而非 private** **未定义函数 (Prefer deleted functions to private undefined ones)**



```
class Widget11 {
public:
	Widget11(const Widget11&) = delete;
	Widget11& operator=(const Widget11&) = delete;
 
	template<typename T>
	void processPointer(T* ptr) {}
};
 
template<>
void Widget11::processPointer<void>(void*) = delete;
 
bool isLucky(int number) { return false; } // 原始版本
bool isLucky(char) = delete; // 拒绝char型别
bool isLucky(bool) = delete; // 拒绝bool型别
bool isLucky(double) = delete; // 拒绝double和float型别
 
template<typename T>
void processPointer(T* ptr) {}
template<>
void processPointer<void>(void*) = delete; // 不可以使用void*来调用processPointer
template<>
void processPointer<char>(char*) = delete; // 不可以使用char*来调用processPointer
 
int test_item_11()
{
	//if (isLucky('a')) {} // error, call to deleted function
 
	return 0;
}
```



C++98 中为了阻止个别成员函数的使用采取的做法是声明其为 private，并且不去定义它们。在 C++11 中，有更好的途径来达成效果上相同的结果：使用”=delete。删除函数无法通过任何方法使用。**习惯上，删除函数会被声明为** **public****，而非 private**。任何函数都能成为删除函数 (any function may be deleted)，但只有成员函数能声明为 private。还有一个妙处是删除函数能做到而 private 成员函数做不到的，那就是阻止那些不应该进行的模板具现 (template instantiation)。



指针世界中有两个异类：一个是 void * 指针，因为无法对其执行提领 (dereference)、自增、自减等操作。还有一个是 char * 指针，因为它们基本上表示的是 C 风格的字符串，而不是指涉到单个字符的指针。



**要点速记：(1).** **优先选用删除函数 (deleted function)****，而非 private** **未定义函数。(2).** **任何函数都可以 deleted****，包括非成员函数和模板具现 (template instantiation)**。



“= delete;” 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52475108



## **12.** **为意在改写的函数添加 override** **声明 (Declare overriding functions override)**



```
class Base {
public:
	virtual void doWork() {} // 基类中的虚函数
};
 
class Derived : public Base {
public:
	virtual void doWork() override {} // 改写(override)了Base:doWork(“virtual”在这可写可不写)
};
 
class Widget12 {
public:
	void doWork() & { fprintf(stdout, "&\n"); } // 这个版本的doWork仅在*this是左值时调用
	void doWork() && { fprintf(stdout, "&&\n"); } // 这个版本的doWork仅在*this是右值时调用
 
	using DataType = std::vector<double>;
	DataType& data() & { fprintf(stdout, "data() &\n"); return values; } // 对于左值Widget12型别，返回左值
	DataType data() && { fprintf(stdout, "data() &&\n"); return std::move(values); } // 对于右值Widget12型别，返回右值
 
private:
	DataType values;
};
 
Widget12 makeWidget() // 工厂函数(返回右值)
{
	Widget12 w;
	return w;
}
 
void doSomething(Widget12& w) {} // 仅接受左值的Widget12型别
void doSomething(Widget12&& w) {} // 仅接受右值的Widget12型别
 
int test_item_12()
{
	std::unique_ptr<Base> upb = std::make_unique<Derived>(); // 创建基类指针，指涉到派生类对象
	upb->doWork(); // 通过基类指针调用doWork,结果是派生类函数被调用
 
	Widget12 w; // 普通对象(左值)
	w.doWork(); // 以左值调用Widget12::doWork(即Widget12::doWork &)
	makeWidget().doWork(); // 以右值调用Widget12::doWork(即Widget12::doWork &&)
 
	auto vals1 = w.data(); // 调用Widget12::data的左值重载版本,vals1拷贝构造完成初始化
	auto vals2 = makeWidget().data(); // 调用Widget12::data的右值重载版本，vals2采用移动构造完成初始化
 
	return 0;
}
```



如果要改写 (override) 真的发生，有一系列要求必须满足：(1). 基类中的函数必须是虚函数。(2). 基类和派生类中的函数名字必须完全相同 (析构函数例外)。(3). 基类和派生类中的函数形参型别必须完全相同。(4). 基类和派生类中的函数常量性(constness) 必须完全相同。(5). 基类和派生类中的函数返回值和异常规格 (exception specification) 必须兼容。除了 C++98 给出的这些限制，C++11 又加了一条。(6). 基类和派生类中的函数引用饰词 (reference qualifier) 必须完全相同。引用饰词是为了实现限制成员函数仅用于左值或右值。带有引用饰词的成员函数，不必是虚函数。



C++11 提供了一种方法来显式地标明派生类中的函数是为了改写 (override) 基类版本：为其加上 override 声明。



成员函数引用饰词的作用就是针对发起成员函数调用的对象，即 * this，加一些区分度。这和在成员函数声明末尾加一个 const 的情形一模一样：后者表明发起成员函数调用的对象，即 * this，应为 const。



**要点速记：(1).** **为意在改写的函数添加 override** **声明。(2).** **成员函数引用饰词使得对于左值和右值对象 (\*this)** **的处理能够区分开来**。



override 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52304284



## **13.** **优先选用 const_iterator****，而非 iterator(Prefer const_iterators to iterators)**



```
template<class C>
auto cbegin_(const C& container) -> decltype(std::begin(container)) // cbegin的一个实现
{
	return std::begin(container);
}
 
template<class C>
auto cend_(const C& container) -> decltype(std::end(container)) // cend的一个实现
{
	return std::end(container);
}
 
int test_item_13()
{
	std::vector<int> values{1, 10, 1000};
	auto it = std::find(values.cbegin(), values.cend(), 1983); // use cbegin and cend
	values.insert(it, 1998);
 
#ifdef _MSC_VER
	auto it2 = std::find(std::cbegin(values), std::cend(values), 1983); // C++14,非成员函数版本的cbegin, cend, gcc 4.9.4 don't support
	values.insert(it2, 1998);
#endif
 
	auto it3 = std::find(cbegin_(values), cend_(values), 1983);
	values.insert(it3, 1998);
 
	return 0;
}
```



const_iterator 是 STL 中相当于指涉到 const 的指针的等价物。它们指涉到不可被修改的值。**任何时候只要你需要一个迭代器而其指涉到的内容没有修改必要，你就应该使用** **const_iterator**。



**要点速记：(1).** **优先选用 const_iterator****，而非 iterator****。(2).** **在最通用的代码中，优先选用非成员函数版本的 begin****、end** **和 rbegin** **等，而非其成员函数版本**。



## **14.** **只要函数不会发射异常，就为其加上 noexcept** **声明 (Declare functions noexcept if they won’t emit exceptions)**



```
int f14_1(int x) throw() { return 1; } // f14_1不会发射异常: C++98风格
int f14_2(int x) noexcept { return 2; } // f14_2不会发射异常: C++11风格
 
//RetType function(params) noexcept; // 最优化
//RetType function(params) throw(); // 优化不够
//RetType function(params); // 优化不够
 
int test_item_14()
{
	return 0;
}
```



在 C++11 中，无条件的 noexcept 就是为了不会发射异常 (emit exception) 的函数而准备的。函数是否带有 noexcept 声明，就和成员函数是否带有 const 声明是同等重要的信息。当你明明知道一个函数不会发射异常却未给它加上 noexcept 声明的话，这就是接口规格缺陷。对不会发射异常的函数应用 noexcept 声明还有一个动机，那就是它可以让编译器生成更好的目标代码。



在带有 noexcept 声明的函数中，优化器不需要在异常传出函数的前提下，将执行期栈 (runtime stack) 保持在可开解状态 (unwindable state)；也不需要在异常溢出函数的前提下，保证所有其中的对象以其被构造顺序的逆序完成析构。而那些以”throw()” 异常规格 (exception specification) 声明的函数就享受不到这样的优化灵活性，和没有加异常规格声明的函数一样。



**要点速记：(1).noexcept** **声明是函数接口的组成部分，这意味着调用方可能会对它有依赖。(2).** **相对于不带 noexcept** **声明的函数，带有 noexcept** **声明的函数有更多的机会得到优化。(3).noexcept** **性质对于移动操作、swap****、内存释放函数和析构函数最有价值。(4).** **大多数函数都是异常中立的，不具备 noexcept** **性质**。



## **15.** **只要有可能使用 constexpr****，就使用它 (Use constexpr whenever possible)**



```
// pow前面写的那个constexpr并不表明pow要返回一个const值，它表明的是如果base和exp是编译期常量，pow的返回结果
// 就可以当一个编译期常量使用；如果base和exp中有一个不是编译期常量，则pow的返回结果就将在执行期计算
constexpr int pow(int base, int exp) noexcept // pow is a constexpr func that never throws
{
	return (exp == 0 ? 1 : base * pow(base, exp - 1)); // C++11
	//auto result = 1; // C++14
	//for (int i = 0; i < exp; ++i) result *= base;
	//return result;
}
 
auto readFromDB(const std::string& str) { return 1; }
 
class Point15 {
public:
	constexpr Point15(double xVal = 0, double yVal = 0) noexcept : x(xVal), y(yVal) {}
	constexpr double xValue() const noexcept { return x; }
	constexpr double yValue() const noexcept { return y; }
	void setX(double newX) noexcept { x = newX; }
	//constexpr void setX(double newX) noexcept { x = newX; } // C++14
	void setY(double newY) noexcept { y = newY; }
	//constexpr void setY(double newY) noexcept { y = newY; } // C++14
private:
	double x, y;
};
 
constexpr Point15 midpoint(const Point15& p1, const Point15& p2) noexcept
{
	return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2}; // call constexpr member function
}
 
int test_item_15()
{
	int sz = 0; // non-constexpr variable
	//constexpr auto arraySize1 = sz; // error, sz's value not known at compilation
	//std::array<int, sz> data1; // error, sz's value not known at compilation
	constexpr auto arraySize2 = 10; // fine, 10 is a compile-time constant
	std::array<int, arraySize2> data2; // fine, arraySize2 is constexpr
 
	// 注意：const对象不一定经由编译器已知值来初始化
	const auto arraySize3 = sz; // fine, arraySize3 is const copy of sz,arraySize3是sz的一个const副本
	//std::array<int arraySize3> data3; // error, arraySize3.s value not known at compilation
 
	constexpr auto numConds = 5;
	std::array<int, pow(3, numConds)> results; // results has 3^numConds elements
 
	auto base = readFromDB("base"); // get these values at runtime
	auto exp = readFromDB("exponent");
	auto baseToExp = pow(base, exp); // call pow function at runtime
 
	constexpr Point15 p1(9.4, 27.7); // fine, "runs" constexpr constructor during compilation
	constexpr Point15 p2(28.8, 5.3);
 
	constexpr auto mid = midpoint(p1, p2); // 使用constexpr函数的结果来初始化constexpr对象
 
	return 0;
}
```



当 constexpr 应用于对象时，其实就是一个加强版的 const。但应用于函数时，你既不能断定它是 const，也不能假定其值在编译阶段就已知。



所有 constexpr 对象都是 const 对象，但并非所有的 const 对象都是 constexpr 对象。如果你想让编译器提供保证，让变量拥有一个值，用于要求编译期常量的语境，那么能达到这个目的的工具是 constexpr，而非 const。



constexpr 函数可以用在要求编译期常量的语境中。在这样的语境中，若你传给一个 constexpr 函数的实参值是在编译期已知的，则结果也会在编译期计算出来。如果任何一个实参值在编译期未知，则你的代码将无法通过编译。在调用 constexpr 函数时，若传入的值有一个或多个在编译期未知，则它的运作方式和普通函数无异，亦即它也是在运行期执行结果的计算。



**在 C++11** **中，constexpr** **函数不得包含多于一个可执行语句，即一条 return** **语句**。在 C++14 中没有这样的限制。



**要点速记：(1).constexpr** **对象都具备 const** **属性，并由编译期已知的值完成初始化。(2).constexpr** **函数在调用时若传入的实参值是编译期已知的，则会产生编译期结果。(3).** **比起非 constexpr** **对象或非 constexpr** **函数而言，constexpr** **对象或 constexpr** **函数可以用在一个作用域更广的语境中。(4).constexpr** **是对象或函数接口的一部分**。



## **16.** **保证 const** **成员函数的线程安全性 (Make const member functions thread safe)**



```
class Point16 { // 使用std::atomic型别的对象来计算调用次数
public:
	double distanceFromOrigin() const noexcept
	{
		++callCount; // 带原子性的自增操作
		return std::sqrt((x*x) + (y*y));
	}
 
private:
	mutable std::atomic<unsigned> callCount{0};
	double x, y;
};
 
class Widget16 {
public:
	int magicValue() const
	{
		std::lock_guard<std::mutex> guard(m); // lock m
		if (cacheValid) return cachedValue;
		else {
			auto val1 = expensiveComputation1();
			auto val2 = expensiveComputation2();
			cachedValue = val1 + val2;
			cacheValid = true;
			return cachedValue;
		}
	} // unlock m
 
private:
	int expensiveComputation1() const { return 1; }
	int expensiveComputation2() const { return 2; }
 
	mutable std::mutex m;
	mutable int cachedValue; // no longer atomic
	mutable bool cacheValid{false};
};
 
int test_item_16()
{
	return 0;
}
```



对于单个要求同步的变量或内存区域, 使用 std::atomic 就足够了。但是如果有两个或更多个变量或内存区域需要作为一整个单位进行操作时，就要动用互斥量了。



std::mutex 是个只移型别 (std::mutex is a move-only type)(i.e., a type that can be moved, but not copied)。与 std::mutex 一样，std::atomic 也是只移型别。



**要点速记：(1).** **保证 const** **成员函数的线程安全性，除非可以确信它们不会用在并发语境中。(2).** **运用 std::atomic** **型别的变量会比运用互斥量提供更好的性能，但前者仅适用对单个变量或内存区域的操作**。



## **17.** **理解特种成员函数的生成机制 (Understand special member function generation)**



```
class Widget17 {
public:
	Widget17(Widget17&& rhs); // move constructor
	Widget17& operator=(Widget17&& rhs); // move assignment operator
 
	Widget17(const Widget17&) = default; // default copy constructor, behavior is OK
	Widget17& operator=(const Widget17&) = default; // default copy assign, behavior is OK
};
 
int test_item_17()
{
	return 0;
}
```



在 C++ 官方用语中，特种成员函数是指那些 C++ 会自行生成的成员函数。C++98 有四种特种成员函数：**默认构造函数、析构函数、拷贝构造函数、以及拷贝赋值运算符。这些函数仅在需要时才会生成**，亦即，在某些代码使用了它们，而在类中并未显式声明的场合。仅当一个类没有声明任何构造函数时，才会生成默认构造函数 (只要指定了一个要求传参的构造函数，就会阻止编译器生成默认构造函数)。生成的特种成员函数都具有 public 访问层级且是 inline 的，而且它们都是非虚的，除非讨论的是一个析构函数，位于一个派生类中，并且基类的析构函数是个虚函数。在那种情况下，编译器为派生类生成的析构函数也是个虚函数。



在 C++11 中，特种成员函数加入了两位新成员：移动构造函数和移动赋值运算符。移动操作也仅在需要时才生成，而一旦生成，它们执行的也是作用于非静态成员的”按成员移动”操作。意思是，移动构造函数将依照其形参 rhs 的各个非静态成员对于本类的对应成员执行移动构造，而移动赋值运算符则将依照其形参 rhs 的各个非静态成员对于本类的对应成员执行移动赋值。移动构造函数同时还会移动构造它的基类部分 (如果有的话)，而移动赋值运算符则会移动赋值它的基类部分。不过，当我提到移动操作在某个数据成员或基类部分上执行移动构造或移动赋值的时候，并不能保证移动操作真的会发生。” 按成员移动 (Memberwise moves)” 实际上更像是按成员的移动请求，因为那些不可移动的型别 (即那些并未为移动操作提供特殊支持的型别，这包括了大多数 C++98 中的遗留型别) 将通过其拷贝操作实现”移动”。



两种拷贝操作是彼此独立的 (the two copy operations are independent)：声明了其中一个，并不会阻止编译器生成另一个。两种移动操作并不彼此独立 (the two move operations are not independent)：声明了其中一个，就会阻止编译器生成另一个。此外，一旦显式声明了拷贝操作，这个类也就不再会生成移动操作了。反之亦然，一旦声明了移动操作 (无论是移动构造还是移动赋值)，编译器就会废除拷贝操作 (废除的方式是删除它们)。



移动操作的生成条件 (如果需要生成) 仅当以下三者同时成立：该类未声明任何拷贝操作。该类未声明任何移动操作。该类未声明任何析构函数。



C++11 中，支配特种成员函数的机制如下：(1). 默认构造函数：与 C++98 的机制相同。仅当类中不包含用户声明的构造函数时才生成。(2). 析构函数：与 C++98 的机制基本相同，唯一的区别在于析构函数默认为 noexcept。与 C++98 的机制相同，仅当基类的析构函数为虚的，派生类的析构函数才是虚的。(3). 拷贝构造函数：运行期行为与 C++98 相同：按成员进行非静态数据成员的拷贝构造。仅当类中不包含用户声明的拷贝构造函数时才生成。如果该类声明了移动操作，则拷贝构造函数将被删除。在已经存在拷贝赋值运算符或析构函数的条件下，仍然生成拷贝构造函数已经成为了被废弃的行为。(4). 拷贝赋值运算符：运行期行为与 C++98 相同：按成员进行非静态数据成员的拷贝赋值。仅当类中不包含用户声明的拷贝赋值运算符时才生成。如果该类声明了移动操作，则拷贝构造函数将被删除。在已经存在拷贝构造函数或析构函数的条件下，仍然生成拷贝赋值运算符已经成为了被废弃的行为。(5). 移动构造函数和移动赋值运算符：都按成员进行非静态数据成员的移动操作。仅当类中不包含用户声明的拷贝操作、移动操作和析构函数时才生成。



**要点速记：(1).** **特种成员函数是指那些 C++** **会自行生成的成员函数：默认构造函数、析构函数、拷贝操作、以及移动操作。(2).** **移动操作仅当类中未包含用户显式声明的拷贝操作、移动操作和析构函数时才生成。(3).** **拷贝构造函数仅当类中不包含用户显式声明的拷贝构造函数时才生成，如果该类声明了移动操作则拷贝构造函数将被删除。拷贝赋值运算符仅当类中不包含用户显式声明的拷贝赋值运算符才生成，如果该类声明了移动操作则拷贝赋值运算符将被删除。在已经存在显式声明的析构函数的条件下，生成拷贝操作已经成为了被废弃的行为。(4).** **成员函数模板在任何情况下都不会抑制特种成员函数的生成**。



## **18.** **使用 std::unique_ptr** **管理具备专属所有权的资源 (Use std::unique_ptr for exclusive-ownership resource management)**



```
class Investment { // 投资
public:
	virtual ~Investment() {}
};
class Stock : public Investment {}; // 股票
class Bond : public Investment {}; // 债券
class RealEstate : public Investment {}; // 不动产
 
void makeLogEntry(Investment*) {}
 
auto delInvmt1 = [](Investment* pInvestment) { // custom deleter(a lambda expression), 使用无状态lambda表达式作为自定义析构器
			makeLogEntry(pInvestment);
			delete pInvestment;
		};
 
void delInvmt2(Investment* pInvestment) // 使用函数作为自定义析构器
{
	makeLogEntry(pInvestment);
	delete pInvestment;
}
 
template<typename... Ts>
//std::unique_ptr<Investment> makeInvestment(Ts&&... params) // return std::unique_ptr to an object created from the given args
std::unique_ptr<Investment, decltype(delInvmt1)> makeInvestment(Ts&&... params) // 改进的返回类型,返回值尺寸与Investment*相同
//std::unique_ptr<Investment, void(*)(Investment*)> makeInvestment(Ts&&... params) // 返回值尺寸等于Investment*的尺寸+至少函数指针的尺寸
{
	//std::unique_ptr<Investment> pInv(nullptr);
	std::unique_ptr<Investment, decltype(delInvmt1)> pInv(nullptr, delInvmt1); // ptr to be returned
 
	if (nullptr/* a Stoc object should be created*/) {
		pInv.reset(new Stock(std::forward<Ts>(params)...));
	} else if (nullptr/*a Bond object should be created*/) {
		pInv.reset(new Bond(std::forward<Ts>(params)...));
	} else if (nullptr/*a RealEstate object should be created*/) {
		pInv.reset(new RealEstate(std::forward<Ts>(params)...));
	}
 
	return pInv;
}
 
int test_item_18()
{
	//auto pInvestment = makeInvestment(arguments); // pInvestment is of type std::unique_ptr<Investment>
	//std::shared_ptr<Investment> sp = makeInvestment(arguments); // converts std::unique_ptr to std::shared_ptr
 
	return 0;
} // destroy *pInvestment
```



C++11 中共有四种智能指针：std::auto_ptr、std::unique_ptr、std::shared_ptr 和 std::weak_ptr。所有这些智能指针都是为管理动态分配对象的生命期而设计的，换言之，通过保证这样的对象在适当的时机以适当的方式析构 (包括发生异常的场合)，来防止资源泄漏。



std::auto_ptr 是个从 C++98 中残留下来的弃用特性，它是一种对智能指针进行标准化的尝试，这种尝试后来成为了 C++11 中的 std::unique_ptr。



std::unique_ptr 可以做 std::auto_ptr 能够做的任何事，并且不止于此。它执行的效率和 std::auto_ptr 一样高，而且不用扭曲其要表达的本意去复制任何对象。它从任何方面来看都要比 std::auto_ptr 更好。



每当你需要使用智能指针时，std::unique_ptr 基本上应是手头首选。可以认为在默认情况下 (使用默认析构器)std::unique_ptr 和裸指针(raw pointer) 有着相同的尺寸，并且对于大多数的操作(包括提领(including dereferenceing))，它们都是精确地执行了相同的指令。



std::unique_ptr 实现的是专属所有权 (exclusive ownership) 语义。一个非空的 std::unique_ptr 总是拥有其所指涉到的资源。移动一个 std::unique_ptr 会将所有权从源指针移至目标指针 (源指针被置空)。std::unique_ptr 不允许复制(copy)，因为如果复制了一个 std::unique_ptr，就会得到两个指涉到同一资源的 std::unique_ptr，而这两者都认为自己拥有(因此应当析构) 该资源。因而 std::unique_ptr 是个只移型别(move-only type)。在执行析构操作时，由非空的 std::unique_ptr 析构其资源。默认地，资源的析构是通过对 std::unique_ptr 内部的裸指针实施 delete 完成的。



std::unique_ptr 的一个常见用法是在对象继承谱系中作为工厂函数的返回型别。



默认地，析构通过 delete 运算符实现，但是在析构过程中 std::unique_ptr 可以被设置为使用自定义析构器 (custom delete)：析构资源所调用的任意函数 (或函数对象，包括那些由 lambda 表达式产生的)。



std::unique_ptr 以两种形式提供，一种是单个对象 (std::unique_ptr<T>)，另一种是数组 (std::unique_ptr<T[]>)。单个对象形式不提供索引运算符 (operator[])，而数组形式则不提供提领运算符 (lack dereferencing operator)(operator * 和 operator->)。



std::unique_ptr 是 C++11 中表达专属所有权的方式，但它还有一个十分吸引人的特性，就是 std::unique_ptr 可以方便高效地转换成 std::shared_ptr。



**要点速记：(1).std::unique_ptr** **是小巧、高速的、具备只移型别的智能指针，对托管资源实施专属所有权语义。(2).** **默认地，资源析构采用 delete** **运算符来实现，但可以指定自定义删除器。有状态的删除器和采用函数指针实现的删除器会增加 std::unique_ptr** **型别的对象尺寸。(3).** **将 std::unique_ptr** **转换成 std::shared_ptr** **是容易实现的**。



std::unique_ptr 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52203664



## **19.** **使用 std::shared_ptr** **管理具备共享所有权的资源 (Use std::shared_ptr for shared-ownership resource management)**



```
class Widget19 {};
void makeLogEntry(Widget19*) {}
auto loggingDel = [](Widget19* pw) { makeLogEntry(pw); delete pw; }; // custom deleter,自定义析构器
 
int test_item_19()
{
	std::unique_ptr<Widget19, decltype(loggingDel)> upw(new Widget19, loggingDel); // 析构器型别是智能指针型别的一部分
	std::shared_ptr<Widget19> spw(new Widget19, loggingDel); // 析构器型别不是智能指针型别的一部分
 
	auto pw = new Widget19; // pw是个裸指针
	//std::shared_ptr<Widget19> spw1(pw, loggingDel); // 为*pw创建一个控制块
	//std::shared_ptr<Widget19> spw2(pw, loggingDel); // 为*pw创建了第二个控制块
	// 以上两行语句会导致*pw被析构两次，第二次析构将会引发未定义行为,不推荐上面的用法
 
	std::shared_ptr<Widget19> spw1(new Widget19, loggingDel); // 直接传递new表达式
	std::shared_ptr<Widget19> spw2(spw1); // spw2使用的是和spw1同一个控制块
 
	return 0;
}
```



std::shared_ptr 这种智能指针访问的对象采用共享所有权来管理其生存期。没有哪个特定的 std::shared_ptr 拥有该对象。取而代之的是，所有指涉到它的 std::shared_ptr 共同协作，确保在不再需要该对象的时刻将其析构。当最后一个指涉到某对象的 std::shared_ptr 不再指涉到它时 (例如，由于该 std::shared_ptr 被析构，或使其指涉到另一个不同的对象)，该 std::shared_ptr 会析构其指涉到的对象。



std::shared_ptr 可以通过访问某资源的引用计数来确定是否自己是最后一个指涉到该资源的。引用计数是个与资源关联的值，用来记录跟踪指涉到该资源的 std::shared_ptr 数量。 std::shared_ptr 的构造函数会使该计数递增 (通常如此)，而其析构函数会使该计数递减，而拷贝赋值运算符同时执行两种操作(如果 sp1 和 sp2 是指涉到不同对象的 std::shared_ptr，则赋值运算”sp1=sp2” 将修改 sp1，使其指涉到 sp2 所指涉到的对象。该赋值的净效应是：最初 sp1 所指涉到的对象的引用计数递减，同时 sp2 所指涉到的对象的引用计数递增)。如果某个 std::shared_ptr 发现，在实施过一次递减后引用计数变成了零，即不再有 std::shared_ptr 指涉到该资源，则 std::shared_ptr 会析构之。



引用计数的存在会带来一些性能影响：(1).std::shared_ptr 的尺寸是裸指针的两倍。因为它们内部既包含一个指涉到该资源的裸指针，也包含一个指涉到该资源的引用计数的裸指针。(2). 引用计数的内存必须动态分配。std::shared_ptr 若是由 std::make_ptr 创建，可以避免动态分配的成本。然而仍有一些场景下，不可以使用 std::make_ptr。但无论是不是使用 std::make_ptr，引用计数都会作为动态分配的数据来存储。(3). 引用计数的递增和递减必须是原子操作。因为在不同的线程中可能存在并发的读写器。



从一个已有 std::shared_ptr 移动构造一个新的 std::shared_ptr 会将源 std::shared_ptr 置空，这意味着一旦新的 std::shared_ptr 产生后，原有的 std::shared_ptr 将不再指涉到其资源，结果是不需要进行任何引用计数操作。因此，移动 std::shared_ptr 比拷贝它们要快：拷贝要求递增引用计数，而移动则不需要。这一点对于构造和赋值操作同样成立，所以，移动构造函数比拷贝构造函数快，移动赋值比拷贝赋值快。



与 std::unique_ptr 类似，std::shared_ptr 也使用 delete 运算符作为其默认资源析构机制，但它同样支持自定义析构器。然而这种支持的设计却与 std::unique_ptr 有所不同。对于 std::unique_ptr 而言，析构器的型别是智能指针型别的一部分。但对于 std::shared_ptr 而言，却并非如此。与 std::unique_ptr 的另一点不同，是自定义析构器不会改变 std::shared_ptr 的尺寸。无论析构器是怎样的型别，std::shared_ptr 对象的尺寸都相当于裸指针的两倍。



每一个由 std::shared_ptr 管理的对象都有一个控制块。除了包含引用计数之外，如果该自定义析构器被指定的话，该控制块还包含自定义析构器的一个拷贝。如果指定了一个自定义内存分配器，控制块也会包含一份它的拷贝。控制块还有可能包含其它附加数据，如被称为弱计数的次级引用计数。



一个对象的控制块由创建首个指涉到该对象的 std::shared_ptr 的函数来确定。控制块的创建遵循以下规则：(1).std::make_shared 总是创建一个控制块。(2). 从具备专属所有权的指针 (即 std::unique_ptr 或 std::auto_ptr 指针) 出发构造一个 std::shared_ptr 时，会创建一个控制块。专属所有权指针不使用控制块。(3). 当 std::shared_ptr 构造函数使用裸指针作为实参来调用时，它会创建一个控制块。



尽可能避免将裸指针传递给一个 std::shared_ptr 的构造函数。常用的替代手法，是使用 std::make_shared。如果必须将一个裸指针传递给 std::shared_ptr 的构造函数，就直接传递 new 运算符的结果，而非传递一个裸指针变量。



当你希望一个托管到 std::shared_ptr 的类能够安全地由 this 指针创建一个 std::shared_ptr 时，可以使用 std::enable_shared_from_this。std::enable_shared_from_this 是一个基类模板，其型别形参总是其派生类的类名。std::enable_shared_from_this 定义了一个成员函数，它会创建一个 std::shared_ptr 指涉到当前对象，但同时不会重复创建控制块。这个成员函数的名字是 shared_from_this，每当你需要一个和 this 指针指涉到相同对象的 std::shared_ptr 时，都可以在成员函数中使用它。



std::shared_ptr 不能处理数组。std::shared_ptr 的 API 仅被设计用来处理指涉到单个对象的指针，并没有所谓的 std::shared_ptr<T[]>。



**要点速记：(1).std::shared_ptr** **提供方便的手段，实现了任意资源在共享所有权语义下进行生命周期管理的垃圾回收。(2).** **与 std::unique_ptr** **相比，std::shared_ptr** **的尺寸通常是裸指针尺寸的两倍，它还会带来控制块的开销，并要求原子化的引用计数操作。(3).** **默认的资源析构通过 delete** **运算符进行，但同时也支持定制删除器。删除器的型别对 std::shared_ptr** **的型别没有影响。(4).** **避免使用裸指针型别的变量来创建 std::shared_ptr** **指针**。



std::shared_ptr 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52202007



## **20.** **对于类似 std::shared_ptr** **但有可能空悬的指针使用 std::weak_ptr(Use std::weak_ptr for std::shared_ptr-like pointers than can dangle)**



```
class Widget20 {};
 
int test_item_20()
{
	auto spw = std::make_shared<Widget20>(); // spw构造完成后，指涉到Widget20的引用计数置为1
	std::weak_ptr<Widget20> wpw(spw); // wpw和spw指涉到同一个Widget20，引用计数保持为1
	spw = nullptr; // 引用计数变为0,Widget20对象被析构，wpw空悬(dangle)
	if (wpw.expired()) // 若wpw不再指涉到任何对象
	      fprintf(stdout, "wpw doesn't point to an object\n");
 
	std::shared_ptr<Widget20> spw1 = wpw.lock(); // 若wpw失效，则spw1为空
	auto spw2 = wpw.lock(); // 使用auto,同上，若wpw失效，则spw2为空
	if (spw2 == nullptr) fprintf(stdout, "wpw expired\n");
 
	std::shared_ptr<Widget20> spw3(wpw); // 若wpw失效,抛出std::bad_weak_ptr型别的异常
 
	return 0;
}
```



std::weak_ptr 不能提领 (dereferenced)，也不能检查是否为空。这是因为 std::weak_ptr 并不是一种独立的智能指针，而是 std::shared_ptr 的一种扩充。std::weak_ptr 一般是通过 std::shared_ptr 来创建的。当使用 std::shared_ptr 完成初始化 std::weak_ptr 的时刻，两者就指涉到了相同位置，但 std::weak_ptr 并不影响所指涉到的对象的引用计数。



std::weak_ptr 的空悬 (dangle)，也被称作失效 (expired)。



需要一个原子操作来完成 std::weak_ptr 是否失效的校验，以及在未失效的条件下提供对所指涉到的对象的访问。这个操作可以通过由 std::weak_ptr 创建 std::shared_ptr 来实现。该操作有两种形式。一种形式是 std::weak_ptr::lock，它返回一个 std::shared_ptr。如果 std::weak_ptr 已经失效，则 std::shared_ptr 为空。另一种形式是用 std::weak_ptr 作为实参来构造 std::shared_ptr，这样，如果 std::weak_ptr 失效的话，抛出异常。



**要点速记：(1).** **使用 std::weak_ptr** **来代替可能空悬的 std::shared_ptr****。(2).std::weak_ptr** **可能的用武之地包括缓存、观察者列表、以及避免 std::shared_ptr** **指针环路**。



std::weak_ptr 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52203825



## **21.** **优先选用 std::make_unique** **和 std::make_shared****，而非直接使用 new(Prefer std::make_unique and std::make_shared to direct use of new)**



```
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params) // std::make_unique的一个基础版本，不支持数组和自定义析构器
{
	return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
 
class Widget21 {};
 
int test_item_21()
{
	// 使用了new的版本将被创建对象的型别重复写了两遍，但是make系列函数则没有
	auto upw1(std::make_unique<Widget21>()); // 使用make系列函数
	std::unique_ptr<Widget21> upw2(new Widget21); // 不使用make系列函数
	auto spw1(std::make_shared<Widget21>()); // 使用make系列函数
	std::shared_ptr<Widget21> spw2(new Widget21); // 不使用make系列函数
 
	return 0;
}
```



std::make_shared 是 C++11 的一部分，而 std::make_unique 是在 C++14 中才加入标准库的。



std::make_unique 和 std::make_shared 是三个 make 系列函数中的两个。make 系列函数会把一个任意实参集合完美转发 (perfect-forward) 给动态分配内存的对象的构造函数，并返回一个指涉到该对象的智能指针。make 系列函数的第三个是 std::allocate_shared。它的行为和 std::make_shared 一样，只不过它的第一个实参是个用以动态分配内存的分配器对象。



一般使用 std::make_shared 比直接使用 new 好处：(1). 可编写异常安全的代码；(2). 性能的提升。



有一些情景，不能或者不应使用 make 系列函数。例如，所有的 make 系列函数都不允许使用自定义析构器，但是 std::unique_ptr 和 std::shared_ptr 却都有着允许使用自定义析构器的构造函数。



**要点速记：(1).** **相比于直接使用 new** **表达式，make** **系列函数消除了重复代码、改进了异常安全性，并且对于 std::make_shared** **和 std::allocate_shared** **而言，生成的目标代码会尺寸更小、速度更快。(2).** **不适用使用 make** **系列函数的场景包括需要定制删除器，以及期望直接传递大括号初始化物。(3).** **对于 std::shared_ptr****，不建议使用 make** **系列函数的额外场景包括：自定义内存管理的类；内存紧张的系统、非常大的对象、以及存在比指涉到相同对象的 std::shared_ptr** **生存期更久的 std::weak_ptr**。



## **22.** **使用 Pimpl** **习惯用法时，将特殊成员函数的定义放到实现文件中 (When using the Pimpl Idiom, define special member functions in the implementation file)**



```
// 以下代码假设位于gadget.h文件中
typedef struct Gadget { // Gadget is some userdefined type
	int x, y;
	std::string str;
};
 
// 以下代码假设位于widget.h文件中
class Widget22 {
public:
	Widget22();
	// 注意：这里仅声明，不能定义，定义必须放在widget.cpp文件中，因为Impl是个不完整类型
	~Widget22(); // declaration only
 
	// 添加支持移动操作,注意：这里仅声明，不能定义，定义必须放在widget.cpp文件中,因为Impl是个不完整类型
	Widget22(Widget22&& rhs); // declaration only
	Widget22& operator=(Widget22&& rhs); // declaration only
 
	// 添加拷贝操作
	Widget22(const Widget22& rhs); // declaration only
	Widget22& operator=(const Widget22& rhs); // declaration only
 
private:
	// 原数据成员
	//std::string name;
	//std::vector<double> data;
	//Gadget g1, g2, g3;
 
	struct Impl; // declare implementation struct
	//Impl* pImpl; // and pointer to it
	std::unique_ptr<Impl> pImpl; // 使用智能指针代替裸指针(raw pointer)
	// 如果使用std::shared_ptr而非std::unique_ptr，则无需再有析构函数或移动操作的声明
};
 
// 以下代码假设位于widget.cpp中
//#include "widget.h"
//#include "gadget.h"
//#include <string>
//#include <vector>
 
struct Widget22::Impl { // Widget22::Impl的实现，包括此前在Widget22中的数据成员
	std::string name;
	std::vector<double> data;
	Gadget g1, g2, g3;
};
 
//Widget22::Widget22() : pImpl(new Impl) {} // allocate data members for this Widget22 object
Widget22::Widget22() : pImpl(std::make_unique<Impl>()) {}
//Widget22::~Widget22() { delete pImpl; } // destroy data members for this Widget22 object
Widget22::~Widget22() = default; // ~Widget22 definition
Widget22::Widget22(Widget22&& rhs) = default;
Widget22& Widget22::operator=(Widget22&& rhs) = default;
Widget22::Widget22(const Widget22& rhs) : pImpl(std::make_unique<Impl>(*rhs.pImpl)) {}
Widget22& Widget22::operator=(const Widget22& rhs) { *pImpl = *rhs.pImpl; return *this; }
 
int test_item_22()
{
	Widget22 w1;
	auto w2(std::move(w1));
	Widget22 w3(w2);
	w1 = std::move(w2);
 
	return 0;
}
```



“Pimpl”意为”pointer to implementation”，即指涉到实现的指针。这种技巧就是把某类的数据成员用一个指涉到某实现类 (或结构体) 的指针替代，尔后把原来在主类中的数据成员放置到实现类中，并通过指针间接访问这些数据成员。



Pimpl 习惯用法的第一部分，是声明一个指针型别的数据成员，指涉到一个非完整型别 (an incomplete type)。第二部分是动态分配和回收(dynamic allocation and deallocation) 持有从前在原始类里的那些数据成员的对象，而分配和回收代码则放在实现文件中。



**Pimpl** **习惯用法是一种可以在类实现和类使用者之间减少编译依赖性的方法**，但从概念上说，Pimpl 习惯用法并不能改变类所代表的事物。



std::unique_ptr 和 std::shared_ptr 这两种智能指针在实现 pImpl 指针行为时的不同，源自它们对于自定义析构器的支持的不同。对于 std::unique_ptr 而言，析构器型别是智能指针型别的一部分，这使得编译器会产生更小尺寸的运行期数据结构以及更快速的运行期代码。如此高效带来的后果是，预使用编译器产生的特种函数 (例如，析构函数或移动操作)，就要求其指涉到的型别必须是完整型别。而对于 std::shared_ptr 而言，析构器的型别并非智能指针型别的一部分，这就需要更大尺寸的运行时期数据结构以及更慢一些的目标代码，但在使用编译器生成的特种函数时，其指涉到的型别却并不要求是完整型别。



**要点速记：(1).Pimpl** **习惯用法通过降低类的客户和类实现者之间的依赖性，减少了构建遍数。(2).** **对于采用 std::unique_ptr** **来实现的 pImpl** **指针，须在类的头文件中声明特种成员函数，但在实现文件中实现它们。即使默认函数实现有着正确行为，也必须这样做。(3).** **上述建议仅适用于 std::unique_ptr****，但并不适用于 std::shared_ptr**。



## **23.** **理解 std::move** **和 std::forward(Understand std::move and std::forward)**



```
// 比较接近C++11中std::move的示例实现,它不完全符合标准的所有细节
template<typename T> // in namespace std
typename std::remove_reference<T>::type&& move(T&& param)
{
	using ReturnType = typename std::remove_reference<T>::type&&; // 别名声明
	return static_cast<ReturnType>(param);
}
 
// C++14中比较接近的std::move示例实现
template<typename T> // C++14, still in namespace std
decltype(auto) move(T&& param)
{
	using ReturnType = std::remove_reference_t<T>&&;
	return static_cast<ReturnType>(param);
}
 
class Widget23 {};
 
void process(const Widget23& lvalArg)  { fprintf(stderr, "process lvalues\n"); } // process lvalues
void process(Widget23&& rvalArg) { fprintf(stderr, "process rvalues\n"); } // process rvalues
 
template<typename T>
void logAndProcess(T&& param) // template that passes param to process
{
	process(std::forward<T>(param));
}
 
int test_item_23()
{
	Widget23 w;
	logAndProcess(w); // call with lvalue
	logAndProcess(std::move(w)); // call with rvalue
 
	return 0;
}
```



移动语义 (move semantics)：使得编译器得以使用不那么昂贵的移动操作来替换昂贵的拷贝操作 (makes it possible for compilers to replace expensive copying operations with less expensive moves)。同拷贝构造函数、拷贝赋值运算符给予人们控制对象拷贝的具体意义的能力一样，移动构造函数和移动赋值运算符也给予人们控制对象移动语义的能力。移动语义也使得创建只移型别对象成为可能，这些型别包括 std::unique_ptr、std::future 和 std::thread 等。



完美转发 (perfect forwarding)：使得人们可以撰写接受任意实参的函数模板，并将其转发到其它函数，目标函数会接受到与转发函数所接受的完全相同的实参 (makes it possible to write function templates that take arbitrary arguments and forward them to other functions such that the target functions receive exactly the same arguments as were passed to the forwarding functions)。



右值引用是将这两个 (移动语义和完美转发) 风马牛不相及的语言特性胶合起来的底层语言机制，正是它使得移动语义和完美转发成为了可能。



std::move 并不进行任何移动，std::forward 也不进行任何转发。这两者在运行期都无所作为。它们不会生成任何可执行代码，连一个字节都不会生成。(std::move doesn’t move anything. std::forward doesn’t forward anything. At runtime, neither does anything at all. They generate no executable code. Not a single byte.)



**std::move** **和 std::forward** **都是仅仅执行强制型别转换的函数** (其实是函数模板)。std::move 无条件地将实参强制转换成右值，而 std::forward 则仅在某个特定条件满足时才执行同一个强制转换。



std::move 的形参是指涉到一个对象的引用 (准确地说，是万能引用(universal reference))，它返回的是指涉到同一个对象的引用。函数返回值的”&&” 部分，暗示着 std::move 返回的是个右值引用。如果 T 碰巧是个左值引用的话，那么 T&& 就成了左值引用。为了避免这种情况发生，它将型别特征 std::remove_reference 应用于 T，从而保证”&&”应用在一个非引用型别之上。这么一来，就可以确保 std::move 返回的是右值引用，而这一点十分重要，因为从该函数返回的右值引用肯定是右值。综上所述，std::move 将实参强制转换成了右值，而这就是该函数全部的所做作为。



右值是可以实施移动的，所以在一个对象上实施了 std::move，就是告诉编译器该对象具备可移动的条件。右值也仅在通常情况下能够移动。



如果想取得对某个对象执行移动操作的能力，则不要将其声明为常量，因为针对常量对象执行的移动操作将一声不响地变换成拷贝操作。std::move 不仅不实际移动任何东西，甚至不保证经过其强制型别转换后的对象具备可移动的能力。关于针对任意对象实施过 std::move 的结果，唯一可以确定的是，该结果会是个右值。



std::forward 与 std::move 类似，只是与 std::move 会无条件地将其实参强制转换为右值型别不同，std::forward 仅在特定条件下才实施这样的强制型别转换。换言之，std::forward 是一个有条件强制型别转换 (std::forward is a conditional cast)：仅当其实参是使用右值完成初始化时，它才会执行向右值型别的强制型别转换。



使用 std::move 所要传达的意思是无条件地向右值型别的强制型别转换，而使用 std::forward 则想说明仅仅对绑定到右值的引用实施向右值型别的强制型别转换。这是两个非常不同的行为。前者是典型地为移动操作做铺垫，而后者仅仅是传递 (转发) 一个对象到另一个函数 (just passes----forwards----an object to another function)，而在此过程中无论该对象原始型别具备左值性或右值性(lvalueness or rvalueness)，都保持原样。这两个行为是如此不同，因而最好使用两个不同函数(以及函数名字) 来区分这两者。



**要点速记：(1).std::move** **实施的是无条件的向右值型别的强制型别转换。就其本身而言，它不会执行移动操作。(2).** **仅当传入的实参被绑定到右值时，std::forward** **才针对该实参实施向右值型别的强制型别转换。(3).** **在运行期，std::move** **和 std::forward** **都不会做任何操作**。



std::move 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52558914



std::forward 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52589454



## **24.** **区分万能引用和右值引用 (Distinguish universal references from rvalue references)**



```
class Widget24 {};
 
void f24(Widget24&& param) { fprintf(stdout, "Widget24&&\n"); } // no type deduction(不涉及型别推导), param is an rvalue reference
template<typename T>
void f24_1(std::vector<T>&& param) { fprintf(stdout, "std::vector<T>&&\n"); } // param is an rvalue reference
template<typename T>
void f24(T&& param) { fprintf(stdout, "T&&\n"); } // not rvalue reference, param is a universal reference
template<typename T>
void f24(const T&& param) {} // param is an rvalue reference
 
int test_item_24()
{
	Widget24&& var1 = Widget24(); // no type deduction, var1 is an rvalue reference
	auto&& var2 = var1; // not rvalue reference, var2 is a universal reference
 
	Widget24 w;
	f24(w); // lvalue passed to f, param's type is Widget24&(an lvalue reference)
	f24(std::move(w)); // rvalue passed to f, param's type is Widget24&&(an rvalue reference)
 
	std::vector<int> v;
	//f24_1(v); // error, can't bind lvalue to rvalue reference
	f24_1(std::move(v));
	//f24(v); // will call: void f24(T&& param)
 
	return 0;
}
```



实际上，”T&&” 有两种不同的含义。其中一种含义，理所当然，是右值引用。正如期望，它们仅仅会绑定到右值，而其主要的存在理由，在于识别出可移对象。”T&&” 的另一种含义，则表示其既可以是右值引用，亦可以是左值引用，二者居一。带有这种含义的引用在代码中形如右值引用 (即 T&&)，但它们可以像左值引用一样运作 (即 T&)。这种双重特性使之既可以绑定到右值 (如右值引用)，也可以绑定到左值 (即左值引用)。犹有进者 (furthermore)，它们也可以绑定到 const 对象或非 const 对象，以及 volatile 对象或非 volatile 对象，甚至绑定到那些既带有 const 又带有 volatile 饰词的对象。它们几乎可以绑定到万事万物。这种拥有史无前例的灵活性的引用值得拥有一个独特的名字。我称之为万能引用 (universal reference、forwarding reference)。



万能引用会在两种场景下现身。最常见的一种场景是函数模板的形参。第二个场景是 auto 声明。这两个场景的共同之处，在于它们都涉及型别推导 (type deduction)。如果你看到了”T&&”，却没有涉及型别推导，那么，你看到的就是个右值引用。



因为万能引用首先是个引用，所以初始化是必须的。万能引用的初始化物会决定它代表的是个左值引用还是右值引用：如果初始化物是右值，万能引用就会对应到一个右值引用；如果初始化物是左值，万能引用就会对应到一个左值引用。



若要使一个引用成为万能引用，其涉及型别推导是必要条件，但还不是充分条件。引用声明的形式也必须正确无误，并且该形式被限定得很死：**必须得正好形如****”T&&”** **才行**，但没必要一定要取”T” 这个名字。声明为 auto&& 型别的变量都是万能引用，因为它们肯定涉及型别推导并且肯定有正确的形式 (“T&&”)。



**要点速记：(1).** **如果函数模板形参具备 T&&** **型别，并且 T** **的型别系推导而来，或如果对象使用 auto&&** **声明其型别，则该形参或对象就是个万能引用。(2).** **如果型别声明并不精确地具备 type&&** **的形式，或者型别推导并未发生，则 type&&** **就代表右值引用。(3).** **若采用右值来初始化万能引用，就会得到一个右值引用。若采用左值来初始化万能引用，就会得到一个左值引用**。



右值引用更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/78619152



## **25.** **针对右值引用实施 std::move****，针对万能引用实施 std::forward(Use std::move on rvalue references, std::forward on universal references)**



```
class Widget25 {
public:
	Widget25(Widget25&& rhs) : name(std::move(rhs.name)), p(std::move(rhs.p)) {} // rhs is rvalue reference
	template<typename T>
	void setName(T&& newName) { name = std::forward<T>(newName); } // newName is universal reference
 
private:
	std::string name;
	typedef struct SomeDataStructure {} SomeDataStructure;
	std::shared_ptr<SomeDataStructure> p;
};
 
class Matrix {
public:
	Matrix& operator+=(const Matrix& rhs) { return *this; }
	void reduce() {}
};
 
Matrix operator+(Matrix&& lhs, const Matrix& rhs) // 按值返回
{
	lhs += rhs;
	return std::move(lhs); // 将lhs移入返回值
}
 
template<typename T>
Matrix reduceAndCopy(T&& mat) // 按值返回，万能引用形参
{
	mat.reduce();
	return std::forward<T>(mat); // 对于右值是移入返回值;对于左值是拷贝入返回值
}
 
int test_item_25()
{
	return 0;
}
```



右值引用仅会绑定到那些可供移动的对象上 (Rvalue references bind only to objects that are candidates for moving)。



当转发右值引用给其它函数时，应当对其实施向右值的无条件强制型别转换 (通过 std::move)，因为它们一定绑定到右值；而当转发万能引用时，应当对其实施向右值的有条件强制型别转换 (通过 std::forward)，因为它们不一定绑定到右值。应当避免针对右值引用实施 std::forward，针对万能引用使用 std::move 的想法更为糟糕。



**要点速记：(1).** **针对右值引用的最后一次使用实施 std::move****，针对万能引用的最后一次使用实施 std::forward****。(2).** **作为按值返回的函数的右值引用和万能引用，依上一条所述采取相同行为。(3).** **若局部对象可能适用于返回值优化 (RVO, return value optimization)****，则请勿针对其实施 std::move** **或 std::forward**。



## **26.** **避免依万能引用型别进行重载 (Avoid overloading on universal references)**



```
std::multiset<std::string> names; // global data structure
 
//void logAndAdd(const std::string& name) // 第一种实现方法
template<typename T>
void logAndAdd(T&& name) // universal reference，第二种实现方法
{
	auto now = std::chrono::system_clock::now();
	fprintf(stdout, "time point\n");
	//names.emplace(name);
	names.emplace(std::forward<T>(name));
}
 
std::string nameFromIdx(int idx) // 返回索引对应的名字
{
	return std::string("xxx");
}
 
void logAndAdd(int idx) // 新的重载函数
{
	auto now = std::chrono::system_clock::now();
	fprintf(stdout, "time point2\n");
	names.emplace(nameFromIdx(idx));
}
 
int test_item_26()
{
	std::string petName("Darla");
 
	logAndAdd(petName); // as before, copy lvalue into multiset
	logAndAdd(std::string("Persephone")); // move rvalue instead of copying it
	logAndAdd("Patty Dog"); // create std::string in multiset instead of copying a temporary std::string
 
	logAndAdd(22); // 调用形参型别为int的重载版本
 
	short nameIdx = 100;
	//logAndAdd(nameIdx); // error c2664, 形参型别为T&&的版本可以将T推导为short, 对于short型别的实参来说，万能引用产生了比int更好的匹配
 
	return 0;
}
```



形参为万能引用的函数，是 C++ 中最贪婪的，它们会在实例化 (instantiate) 过程中，和几乎任何实参型别都会产生精确匹配。这就是为何把重载和万能引用这两者结合起来总是馊主意：一旦万能引用成为重载候选，它就会吸引走大批的实参型别，远比撰写重载代码的程序员期望的要多。



**要点速记：(1).** **把万能引用作为重载候选型别，几乎总会让该重载版本在始料未及的情况下被调用到。(2).** **完美转发构造函数的问题尤其严重，因为对于非常量的左值型别而言，它们一般都会形成相对于拷贝构造函数的更佳匹配，并且它们还会劫持派生类中对基类的拷贝和移动构造函数的调用**。



## **27.** **熟悉依万能引用型别进行重载的替代方案 (Familiarize yourself with alternatives to overloading on universal references)**



```
std::multiset<std::string> names27; // global data structure
std::string nameFromIdx27(int idx) { return std::string("xxx"); }
 
template<typename T>
void logAndAddImpl(T&& name, std::false_type) // 非整型实参
{
	auto now = std::chrono::system_clock::now();
	fprintf(stdout, "time point: no int\n");
	names27.emplace(std::forward<T>(name));
}
 
void logAndAddImpl(int idx, std::true_type) // 整型实参
{
	auto now = std::chrono::system_clock::now();
	fprintf(stdout, "time point: int\n");
	names.emplace(nameFromIdx(idx));
}
 
template<typename T>
void logAndAdd27(T&& name) // name to data structure
{
	logAndAddImpl(std::forward<T>(name), std::is_integral<typename std::remove_reference<T>::type>());
}
 
class Person {
public:
	//template<typename T, typename = typename std::enable_if<!std::is_same<Person, typename std::decay<T>::type>::value>::type>
	//template<typename T, typename = typename std::enable_if<!std::is_base_of<Person, typename std::decay<T>::type>::value>::type> // 可使继承自Person的类，当调用基类的构造函数时走的是基类的拷贝或移动构造函数
	//template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value>> // C++14
	template<typename T, typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value &&
														!std::is_integral<std::remove_reference_t<T>>::value>> // C++14
	explicit Person(T&& n) // 只有指定的条件满足了才会启用此模板, constructor for string and args convertible to string
		: name(std::forward<T>(n)) 
	{
		// assert that a std::string can be created from a T object
		static_assert(std::is_constructible<std::string, T>::value, "Parameter n can't be used to construct a std::string");
	}
 
	explicit Person(int idx) // constructor for integral args
		: name(nameFromIdx27(idx)) {}
 
private:
	std::string name;
};
 
int test_item_27()
{
	// 注意：test_item_26()与test_item_27()实现的差异
	std::string petName("Darla");
 
	logAndAdd27(petName);
	logAndAdd27(std::string("Persephone"));
	logAndAdd27("Patty Dog");
 
	logAndAdd27(22);
 
	short nameIdx = 100;
	logAndAdd27(nameIdx);
 
	return 0;
}
```



标签分派 (tag dispatch)：型别 std::false_type 和 std::true_type 就是所谓” 标签 (tags)”，运用它们的唯一目的在于强制重载决议(force overload resolution) 按我们想要的方向推进。值得注意的是，这些形参甚至没有名字，它们在运行期不起任何作用。



std::enable_if 可以强制编译器表现出来的行为如同特定的模板不存在一般。这样的模板称为禁用的。默认地，所有的模板都是启用的。可是，实施了 std::enable_if 的模板只会在满足了 std::enable_if 指定的条件的前提下才会启用。



完美转发效率更高，因为它出于和形参声明时的型别严格保持一致的目的，会避免创建临时对象。但是完美转发亦有不足，首先是针对某些型别无法实施完美转发，尽管它们可以被传递到接受特定型别的函数。其次是在客户传递了非法形参时，错误信息的可理解性。



**要点速记：(1).** **使用万能引用和重载组合的替代方案包括使用彼此不同的函数名字、传递 const T&** **型别的形参、传值和标签分派 (Alternatives to the combination of universal references and overloading include the use of distinct function names, passing parameters by lvalue-reference-to-const, passing parameters by value, and using tag dispatch)****。(2).** **通过 std::enable_if** **约束模板允许一起使用万能引用和重载，但是它控制了编译器可以使用万能引用重载的条件。(3).** **万能引用形参通常在性能方面具备优势，但在易用性方面一般会有劣势**。



## **28.** **理解引用折叠 (Understand reference collapsing)**



```
class Widget28 {};
 
template<typename T>
void func(T&& param) {}
 
Widget28 widget28Factory() // function returning rvalue
{
	return Widget28();
}
 
int test_item_28()
{
	Widget28 w; // a variable(an lvalue)
	func(w); // call func with lvalue, T deduced to be Widget28&
 
	func(widget28Factory()); // call func with rvalue, T deduced to be Widget28
 
	auto&& w1 = w; // w1是左值引用
	auto&& w2 = widget28Factory(); // w2是右值引用
 
	return 0;
}
```



你是被禁止声明引用的引用，但编译器却可以在特殊的语境中产生引用的引用，模板实例化就是这样的语境之一。当编译器生成引用的引用时，引用折叠 (reference collapsing) 机制便支配了接下来发生的事情。有两种引用(左值和右值)，所以就有四种可能的引用 -- 引用的组合(左值 -- 左值，左值 -- 右值，右值 -- 左值，右值 -- 右值)。如果引用的引用出现在允许的语境(例如，在模板实例化过程中)，该双重引用会折叠成单个引用，规则如下：如果任一引用为左值引用，则结果为左值引用。否则(即如果两个引用皆为右值引用)，结果为右值引用。引用折叠是使 std::forward 得以运作的关键。



引用折叠会出现的语境有四种：第一种，最常见的一种，就是模板实例化。第二种，是 auto 变量的型别生成。技术细节本质上和模板实例化一模一样，因为 auto 变量的型别推导和模板的型别推导在本质上就是一模一样的。第三种，是生成和使用 typedef 和别名声明。第四种，在于 decltype 的运用中。



万能引用并非一种新的引用型别，其实它就是满足了下面两个条件的语境中的右值引用：(1). 型别推导的过程会区别左值和右值。T 型别的左值推导结果为 T&，而 T 型别的右值则推导结果为 T。(2). 会发生引用折叠。



**要点速记：(1).** **引用折叠会在四种语境中发生：模板实例化、auto** **型别生成、创建和运用 typedef** **和别名声明，以及 decltype****。(2).** **当编译器在引用折叠的语境下生成引用的引用时，结果会变成单个引用。如果原始的引用中有任一引用为左值引用，则结果为左值引用。否则，结果为右值引用。(3).** **万能引用就是在型别推导的过程会区别左值和右值，以及会发生引用折叠的语境中的右值引用 (Universal references are rvalue references in contexts where type deduction distinguishes lvalues from rvalues and where reference collapsing occurs)**。



## **29.** **假定移动操作不存在、成本高、未使用 (Assume that move operations are not present, not cheap, and not used)**



```
class Widget29 {};
 
int test_item_29()
{
 
	std::vector<Widget29> vw1;
	// ... // put data into vw1
	// move vw1 into vw2. runs in constant time. only ptrs in vw1 and vw2 are modified
	auto vw2 = std::move(vw1);
 
	std::array<Widget29, 10000> aw1;
	// ... // put data into aw1
	// move aw1 into aw2. runs in linear time. all elements in aw1 are moved into aw2
	auto aw2 = std::move(aw1);
 
	return 0;
}
```



在下面的几个场景中，C++11 的移动语义不会给你带来什么好处：



(1). 没有移动操作：待移动的对象未能提供移动操作。因此，移动请求就变成了拷贝请求。



(2). 移动未能更快：待移动的对象虽然有移动操作，但并不比其拷贝操作更快。



(3). 移动不可用：移动本可以发生的语境下，要求移动操作不可发射异常 (emit no exception)，但该操作未加上 noexcept 声明。



(4). 源对象是个左值：除了极少数例外，只有右值可以作为移动操作的源。



**要点速记：(1).** **假定移动操作不存在、成本高、未使用。(2).** **对于那些型别或对于移动语义的支持情况已知的代码，则无需作以上假定**。



## **30.** **熟悉完美转发的失败情形 (Familiarize yourself with prefect forwarding failure cases)**



```
void f30(const std::vector<int>& v) {}
void f30_2(std::size_t val) {}
 
template<typename T>
void fwd(T&& param) // accept any argument
{
	f30(std::forward<T>(param)); // forward it to f30
}
 
template<typename T>
void fwd30_2(T&& param) // accept any argument
{
	f30_2(std::forward<T>(param)); // forward it to f30_2
}
 
class Widget30 {
public:
	static const std::size_t MinVals = 28; // MinVals' declaration
};
//const std::size_t Widget30::MinVals; // no define for MinVals
 
struct IPv4Header {
	std::uint32_t version : 4,
		IHL : 4,
		DSCP : 6,
		ECN : 2,
		totalLength : 16;
};
 
int test_item_30()
{
	f30({ 1, 2, 3 }); // fine, "{1, 2, 3}"implicitly converted to std::vector<int>
	//fwd({ 1, 2, 3 }); // error, 大括号初始化物的运用，就是一种完美转发失败的情形
	auto il = { 1, 2, 3 }; // il's type deduced to be std::initializer_list<int>
	fwd(il); // fine, prefect-forwards il to f
 
	std::vector<int> widget30Data;
	widget30Data.reserve(Widget30::MinVals); // use of MinVals
 
	f30_2(Widget30::MinVals); // fine, treated as "f30_2(28)"
	fwd30_2(Widget30::MinVals); // error, shouldn't link, note: windows and linux can link
 
	IPv4Header h;
	memset(&h, 0, sizeof(IPv4Header));
	f30_2(h.totalLength); // fine
 
	//fwd30_2(h.totalLength); // error
	auto length = static_cast<std::uint16_t>(h.totalLength);
	fwd30_2(length); // forward the copy
 
	return 0;
}
```



“转发 (forwarding)” 的含义不过是一个函数把自己的形参传递 (转发) 给另一个函数而已。其目的是为了让第二个函数 (转发目的函数) 接受第一个函数 (转发发起函数) 所接受的同一个对象。这就排除了按值传递形参，因为它们只是原始调用者所传递之物的副本。我们想要转发目的函数能够处理原始传入对象。指针形参也只能出局，因为我们不想强迫调用者传递指针。论及一般意义上的转发时，都是在处理形参为引用型别的情形。



完美转发的含义是我们不仅转发对象，还转发其显著特征：型别、是左值还是右值，以及是否带有 const 或 volatile 饰词等。



**不能实施完美转发的实参**：(1). 大括号初始化物；(2).0 和 NULL 用作空指针：若尝试把 0 和 NULL 以空指针之名传递给模板，型别推导就会发生行为扭曲，推导结果会是整型 (一般情况下会是 int) 而非所传递实参的指针型别。结论就是：0 和 NULL 都不能用作空指针以进行完美转发。不过，修正方案也颇简单：传递 nullptr，而非 0 或 NULL。(3). 仅有声明的整型 static const 成员变量。(4). 重载的函数名字和模板名字。(5). 位域：非 const 引用不得绑定到位域。



**要点速记：(1).** **完美转发的失败情形，是源于模板型别推导失败，或推导结果是错误的型别。(2).** **会导致完美转发失败的实参种类有大括号初始化物、以值 0** **或 NULL** **表达的空指针、仅有声明的整型 static const** **成员变量、模板或重载的函数名字，以及位域**。



## **31.** **避免默认捕获模式 (Avoid default capture modes)**



```
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;
 
class Widget31 {
public:
	void addFilter() const // add an entry to filters
	{
		//filters.emplace_back([=](int value) { return value % divisor == 0; } );
		// 捕获只能针对于在创建lambda式的作用域内可见的非静态局部变量(包括形参)
		//filters.emplace_back([](int value) { return value % divisor == 0; }); // error, divisor not available
		//filters.emplace_back([divisor](int value) { return value % divisor == 0; }); // error, no local divisor to capture
 
		auto currentObjectPtr = this;
		filters.emplace_back([currentObjectPtr](int value) { return value % currentObjectPtr->divisor == 0; });
 
		auto divisorCopy = divisor; // copy data member
		filters.emplace_back([divisorCopy](int value) { return value % divisorCopy == 0; }); // capture the copy use the copy
 
		static int xxx = 2;
		//filters.emplace_back([xxx](int value) { return value % xxx == 0; }); // error
		//filters.emplace_back([=](int value) { return value % xxx == 0; });
		++xxx;
	}
 
private:
	int divisor; // used in Widget31's filter
};
 
int test_item_31()
{
 
	filters.emplace_back(
		[](int value) { return value % 5 == 0; } // 不捕获任何外部变量
	);
	filters.emplace_back(
		[&](int value) { return value % 5 == 0; } // 以引用形式捕获所有外部变量
	);
	filters.emplace_back(
		[=](int value) { return value % 5 == 0; } // 以值的形式捕获所有外部变量
	);
 
	return 0;
}
```



C++11 中有两种默认捕获模式：按引用或按值。



按引用捕获会导致闭包 (closure) 包含指涉到局部变量的引用，或者指涉到定义 lambda 式的作用域内的形参的引用。一旦由 lambda 式所创建的闭包越过了该局部变量或形参的生命期，那么闭包的引用就会空悬(dangle)。显示地列出 lambda 式所依赖的局部变量或形参是更好的软件工程实践。



**捕获只能针对于在创建 lambda** **式的作用域内可见的非静态局部变量 (****包括形参)**。



**要点速记：(1).** **按引用的默认捕获会导致空悬指针问题 (dangling references)****。(2).** **按值的默认捕获极易受空悬指针影响 (****尤其是 this)****，并会误导人们认为 lambda** **式是自洽的 (lambdas are self-contained)**。



lambda 表达式更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52653313



## **32.** **使用初始化捕获将对象移入闭包 (Use init capture to move objects into closures)**



```
class Widget32 {
public:
	bool isValidated() const { return true; }
	bool isProcessed() const { return true; }
	bool isArchived() const { return true; }
 
private:
};
 
class IsValAndArch { // is validated and archived
public:
	using DataType = std::unique_ptr<Widget32>;
 
	explicit IsValAndArch(DataType&& ptr) : pw(std::move(ptr)) {}
 
	bool operator()() const
	{
		return pw->isValidated() && pw->isArchived();
	}
 
private:
	DataType pw;
};
 
std::vector<double> data32; // 欲移入闭包的对象
 
int test_item_32()
{
	auto pw = std::make_unique<Widget32>();
	auto func = [pw = std::move(pw)]{ return pw->isValidated() && pw->isArchived(); }; // C++14, 采用std::move(pw)初始化闭包类的数据成员
	// pw = std::move(pw): 初始化捕获，位于"="左侧的，在你所指定的闭包类中数据成员的名字，而位于"="右侧的则是初始化表达式
	// "="左右两侧处于不同的作用域。左侧作用域就是闭包类的作用域，而右侧的作用域与定义lambda式的作用域相同
	// "pw = std::move(pw)"表达了"在闭包类中创建一个数据成员pw,然后使用针对局部变量pw实施std::move的结果来初始化该数据成员"
	
	auto func2 = [pw = std::make_unique<Widget32>()]{ return pw->isValidated() && pw->isArchived(); }; // C++14, 闭包类数据成员可以由std::make_unique直接初始化
	auto func7 = std::bind([](const std::unique_ptr<Widget32>& pw) {return pw->isValidated() && pw->isArchived(); }, std::make_unique<Widget32>()); // C++11
 
	auto func3 = IsValAndArch(std::make_unique<Widget32>()); // C++11
 
	auto func4 = [data32 = std::move(data32)]{ /*use of data*/ }; // C++14
	auto func5 = std::bind([](const std::vector<double>& data32) { /*use of data*/ }, std::move(data32)); // 初始化捕获的C++11模拟
	auto func6 = std::bind([](std::vector<double>& data32) mutable {/*use of data*/}, std::move(data32)); // 初始化捕获的C++11模拟，for mutable lambda
 
	return 0;
}
```



使用初始化捕获 (init capture)(C++14)，可以使你指定：(1). 由 lambda 生成的闭包类中数据成员的名字。(2). 一个表达式用于初始化该数据成员。



初始化捕获又被称为广义 lambda 捕获 (generalized lambda capture)。



移动捕获在 C++11 中可以采用以下方法模拟：(1). 把需要捕获的对象移动到由 std::bind 产生的函数对象中。(2). 给到 lambda 式一个指涉到欲” 捕获” 的对象的引用。



移动构造 (move-construct) 一个对象进 C++11 闭包是不可能的，但是移动构造一个对象进绑定对象 (bind object) 则是可能的。在 C++11 中模拟移动捕获 (move-capture) 包括以下步骤：先移动构造一个对象进一个绑定对象，然后按引用把该移动构造所得的对象传递给 lambda 式。因为绑定对象的生命期和闭包相同，所以针对绑定对象中的对象和闭包里的对象可以采用同样方法加以处置。



**要点速记：(1).** **使用 C++14** **的初始化捕获将对象移入闭包。(2).** **在 C++11** **中，可由手工实现的类或 std::bind** **去模拟初始化捕获**。



## **33.** **对 auto&&** **型别的形参使用 decltype****，以 std::forward** **之 (Use decltype on auto&& parameters to std::forward them)**



```
int test_item_33()
{
	auto f1 = [](auto x) { return func33(normalize(x)); };
	auto f2 = [](auto&& param) { return func33(normalize(std::forward<decltype(param)>(param))); };
	auto f3 = [](auto&&... param) { return func33(normalize(std::forward<decltype(param)>(param)...)); };
 
	return 0;
}
```



在 C++14 中泛型 lambda 式 (generic lambda) 可以在形参规格 (parameter specification) 中使用 auto。



## **34.** **优先选用 lambda** **式，而非 std::bind(Prefer lambdas to std::bind)**



```
using Time = std::chrono::steady_clock::time_point; // typedef for a point in time
enum class Sound { Beep, Siren, Whistle };
using Duration = std::chrono::steady_clock::duration; // typedef for a length of time
 
void setAlarm(Time t, Sound s, Duration d) {} // at time t, make sound s for duration d
 
int test_item_34()
{
	// setSoundL("L" for "lambda") is a function object allowing a sound to be specified for a 30-sec alarm to go off an hour after it's set
	auto setSoundL1 = [](Sound s) {
		using namespace std::chrono;
		setAlarm(steady_clock::now() + hours(1), s, seconds(30));
	};
 
	auto setSoundL2 = [](Sound s) {
		using namespace std::chrono;
		using namespace std::literals; // C++14
		setAlarm(steady_clock::now() + 1h, s, 30s); // C++14
	};
 
	setSoundL1(Sound::Siren);
	setSoundL2(Sound::Siren);
 
	using namespace std::chrono;
	using namespace std::literals; // C++14
	using namespace std::placeholders; // needed for use of "_1"
	auto setSoundB1 = std::bind(setAlarm, std::bind(std::plus<>(), steady_clock::now(), 1h), _1, 30s); // C++14
	auto setSoundB2 = std::bind(setAlarm, std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), hours(1)), _1, seconds(30)); // C++11
 
	setSoundB1(Sound::Siren);
	//setSoundB2(Sound::Siren);
 
	using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
 
	return 0;
}
```



之所以优先选用 lambda 式，而非 std::bind，最主要原因是 lambda 式具备更高的可读性。比起 lambda 式，使用 std::bind 的代码可读性更差、表达力更低，运行效率也可能更糟。在 C++14 中，根本没有使用 std::bind 的适当用例。而在 C+11 中，std::bind 仅在两个受限的场合还算有着使用的理由：(1). 移动捕获：C++11 的 lambda 式没有提供移动捕获特性，但可以通过结合 std::bind 和 lambda 式来模拟移动捕获。(2). 多态函数对象：因为绑定对象的函数调用运算符利用了完美转发，它就可以接受任何型别的实参。



**要点速记：(1).lambda** **式比起 std::bind** **而言，可读性更好、表达力更强，可能运行效率也更高。(2).** **仅在 C++11** **中，std::bind** **在实现移动捕获，或是绑定到具备模板化的函数调用运算符的对象的场合中，可能尚有余热可以发挥**。



std::bind 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/52613910



## **35.** **优先选用基于任务而非基于线程的程序设计 (Prefer task-based programming to thread-based)**



```
int doAsyncWork() { return 1; }
 
int test_item_35()
{
	std::thread t(doAsyncWork);
	t.join();
	
	auto fut = std::async(doAsyncWork);
 
	return 0;
}
```



如果你想以异步方式运行函数 doAsyncWork，有两种基本选择：你可以创建一个 std::thread，并在其上运行 doAsyncWork，因此这是基于线程 (thread-based) 的方法。或者你可以把 doAsyncWork 传递给 std::async，这是一种基于任务 (task-based) 的策略。



硬件线程是实际执行计算的线程。现代计算机体系结构会为每个 CPU 内核提供一个或多个硬件线程。软件线程 (又称操作系统线程或系统线程) 是操作系统用以实施跨进程的管理，以及进行硬件线程调度的线程。通常，能够创建的软件线程会比硬件线程要多。std::thread 是 C++ 进程里的对象，用作底层软件线程的句柄。软件线程是一种有限的资源，如果你试图创建的线程数量多于系统能够提供的数量，就会抛出 std::system_error 异常。



比起基于线程编程，基于任务的设计能够分担你手工管理线程的艰辛，而且它提供了一种很自然的方式，让你检查异步执行函数的结果 (即返回值或异常)。但是仍有几种情况下，直接使用线程会更适合，它们包括：(1). 你需要访问底层线程(underlying threading) 实现的 API：C++ 并发 API 通常会采用特定平台的低级 API(lower-level platform specific API)来实现，经常使用的有 pthread 或 Windows 线程库。它们提供的 API 比 C++ 提供的更丰富 (例如，C++11 没有线程优先级的概念)。为了访问底层线程实现的 API，std::thread 通常会提供 native_handle 成员函数，而 std::future(即 std::async 的返回型别) 则没有该功能的对应物。(2). 你需要且有能力为你的应用优化线程用法。(3). 你需要实现超越 C++ 并发 API 的线程技术。



**要点速记：(1).std::thread** **的 API** **未提供直接获取异步运行函数返回值的途径，而且如果那些函数抛出异常，程序就会终止。(2).** **基于线程的程序设计要求手动管理线程耗尽 (thread exhaustion)****、超订 (oversubscription)****、负载均衡 (load balancing)****，以及新平台适配。(3).** **通过使用默认启动策略的 std::async** **进行基于任务的编程，可以为你解决大多数此类问题 (Task-based programming via std::async with the default launch policy handles most of these issues for you)**。



std::async 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104133494



## **36.** **如果异步是必要的，则指定 std::launch::async(Specify std::launch::async if asynchronicity is essential)**



```
void f36()
{
	using namespace std::literals; // for C++14 duration suffixes
	std::this_thread::sleep_for(1s);
	//std::this_thread::sleep_for(std::chrono::seconds(1)); // C++11
}
 
template<typename F, typename... Ts>
inline std::future<typename std::result_of<F(Ts...)>::type> reallyAsync(F&& f, Ts&&... params) // C++11, return future for asynchronous call to f(params...)
{
	return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}
 
template<typename F, typename... Ts>
inline auto reallyAsync2(F&& f, Ts&&... params) // C++14
{
	return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}
 
int test_item_36()
{
	// 下面两个调用有着完全相同的意义
	auto fut1 = std::async(f36); // run f using default launch policy
	auto fut2 = std::async(std::launch::async | std::launch::deferred, f36); // run f either async or dererred
 
	auto fut = std::async(f36);
	using namespace std::literals;
	if (fut.wait_for(0s) == std::future_status::deferred) { // 如果任务被推迟了
		// use wait or get on fut to call f synchronously
	} else { // task isn't deferred
		while (fut.wait_for(100s) != std::future_status::ready) { // 不可能死循环(前提假设f36会结束)
			// task is neither deferred nor ready, so do concurrent work until it's ready
		}
 
		// fut is ready
	}
 
	auto fut3 = std::async(std::launch::async, f36); // launch f asynchronously
 
	auto fut4 = reallyAsync(f36); // 以异步方式运行f，如果std::async会抛出异常reallyAsync也会抛出异常
	auto fut5 = reallyAsync2(f36);
 
	return 0;
}
```



当调用 std::async 来执行一个函数 (或可调用对象) 时，仅仅通过 std::async 来运行，你实际上要求的并非一定会达成异步运行的结果，你要求的仅仅是让该函数以符合 std::async 的启动策略 (launch policy) 来运行。**有两个基本策略**，它们都是用限定作用域的枚举型别 std::launch 中的枚举量来表示的。假设函数 f 要传递给 std::async 执行，则：(1).std::launch::async 启动策略意味着函数 f 必须以异步方式运行，即在不同的线程上执行。(2).std::launch::deferred 启动策略意味着函数 f 只会在 std::async 所返回的期值 (future) 的 get 或 wait 得到调用时才运行，即调用方会阻塞直至 f 运行结束为止，如果 get 或 wait 都没有得到调用，f 是不会运行的。**std::async** **的默认启动策略**，也就是你如果不明确指定一个的话，它采用的并非以上两者中的一种。相反地，它采用的是对二者进行或运算的结果。



以默认启动策略对任务使用 std::async 能正常工作需要满足以下所有条件：(1). 任务不需要与调用 get 或 wait 的线程并发执行。(2). 读或写哪个线程的 thread_local 变量都没有关系。(3). 或者可以给出保证在 std::async 返回的期值 (future) 之上可以调用 get 或 wait，或者可以接受任务可能永不执行。(4). 使用 wait_for 或 wait_unitil 的代码会将任务被推迟的可能性纳入考量(the possibility of deferred status into account)。



你想要确保任务以异步方式执行，实现这一点的方法就是在调用时把 std::launch::async 作为第一个实参传入。



**要点速记：(1).std::async** **的默认启动策略既允许任务以异步方式执行，也允许任务以同步方式执行。(2).** **如此的弹性会导致使用 thread_local** **变量时的不确定性，隐含着任务可能永远不会执行，还会影响运用了基于超时的 wait** **调用的程序逻辑。(3).** **如果异步是必要的，则指定 std::launch::async**。



std::future 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104115489



## **37.** **使 std::thread** **型别对象在所有路径皆不可联结 (Make std::threads unjoinable on all paths)**



```
void f37() {}
 
// 下面这个类允许调用者在销毁ThreadRAII对象(用于std::thread的RAII对象)时是调用join还是detach
class ThreadRAII {
public:
	enum class DtorAction { join, detach };
 
	// 构造函数只接受右值型别的std::thread
	ThreadRAII(std::thread&& t, DtorAction a) : action(a), t(std::move(t)) {}
 
	~ThreadRAII()
	{
		if (t.joinable()) {
			if (action == DtorAction::join) {
				t.join();
			} else {
				t.detach();
			}
		}
	}
 
	// support moving
	ThreadRAII(ThreadRAII&&) = default;
	ThreadRAII& operator=(ThreadRAII&&) = default;
 
	std::thread& get() { return t; }
 
private:
	DtorAction action;
	std::thread t;
};
 
int test_item_37()
{
	ThreadRAII trall(std::thread(f37), ThreadRAII::DtorAction::join);
 
	return 0;
}
```



每个 std::thread 型别对象皆处于两种状态之一：可联结或不可联结 (joinable or unjoinable)。可联结的 std::thread 对应底层(underlying) 以异步方式已运行或可运行的线程。std::thread 型别对象对应的底层线程 (underlying thread) 若处于阻塞或等待调度，则它可联结。std::thread 型别对象对应的底层线程如已运行至结束，则亦认为其可联结。



不可联结的 std::thread 不处于以上可联结的状态。不可联结的 std::thread 型别对象包括：(1). 默认构造的 std::thread：此类 std::thread 没有可以执行的函数，因此也没有对应的执行底层线程。(2). 已移动的 std::thread。(3). 已联结的 std::thread。(4). 已分离 (detached) 的 std::thread。



**针对可联结的 std::thread** **型别对象实施析构会导致程序终止**。



**要点速记：(1).** **使 std::thread** **型别对象在所有路径皆不可联结。(2).** **在析构时调用 join** **可能导致难以调试的性能异常。(3).** **在析构时调用 detach** **可能导致难以调试的未定义行为。(4).** **在成员列表的最后声明 std::thread** **型别对象**。



std::thread 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/73393229



## **38.** **对变化多端的线程句柄析构函数行为保持关注 (Be aware of varying thread handle destructor behavior)**



```
class Widget38 { // Widget38 objects might block in their destructor
public: 
 
private:
	std::shared_future<double> fut;
};
 
int calcValue() { return 1; }
 
int test_item_38()
{
	// this container might block in its destructor, because one or more contained futures could refer to a shared state for a nondeferred task launched via std::async
	std::vector<std::future<void>> futs;
 
	std::packaged_task<int()> pt(calcValue); // 给calcValue加上包装使之能以异步方式运行
	auto fut = pt.get_future(); // get future for pt
	std::thread t(std::move(pt));
	t.join();
	
	return 0;
}
```



针对可联结的 std::thread 型别对象实施析构会导致程序终止。而期值 (future) 的析构函数，有时候行为像是执行了一次隐式 join，有时候行为像是执行了一次隐式 detach，有时候行为像是二者都没有执行，但它从不会导致程序终止。



std::packaged_task 型别对象一经创建，就会运行在线程上 (它也可以经由 std::async 的调用而运行，但是如果你要用 std::async 运行任务，就没有很好的理由再去创建什么 std::packaged_task 型别对象，因为 std::async 能够在调度任务执行之前就做到 std::packaged_task 能够做到的任何事情)。std::packaged_task 不能拷贝。



**要点速记：(1).** **期值 (future)** **的析构函数在常规情况下，仅会析构期值的成员变量。(2).** **指涉到经由 std::aysnc** **启动的未推迟 (non-deferred)** **任务的共享状态的最后一个期值会保持阻塞直至该任务结束 (The final future referring to a shared state for a non-deferred task launched via std::async blocks until the task completes)**。



std::packaged_task 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104127352



std::shared_future 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104118831



## **39.** **考虑针对一次性事件通信使用以 void** **为模板型别实参的期值 (Consider void futures for one-shot event communication)**



```
std::promise<void> p;
 
void react() {} // function for reacting task
 
void detect() // function for detecting task, 暂停线程一次
{
	std::thread t([] {p.get_future().wait(); react(); }); // create thread, suspend t until future is set
 
	// ... // here, t is suspended prior to call to react
 
	p.set_value(); // unsuspend t (and thus call react)
 
	// ... // do additional work
 
	t.join(); // make t unjoinable
}
 
void detect_multi() // now for multiple reacting tasks
{
	auto sf = p.get_future().share(); // sf's type is std::shared_future<void>
	std::vector<std::thread> vt; // container for reacting threads
	for (int i = 0; i < /*threadsToRun*/2; ++i) {
		vt.emplace_back([sf] {sf.wait(); react(); }); // wait on local copy of sf
	}
 
	// ... // detect hangs if this "…" code throws
 
	p.set_value(); // unsuspend all threads
 
	// ...
 
	for (auto& t : vt) { // make all threads unjoinable
		t.join();
	}
}
 
int test_item_39()
{
	return 0;
}
```



**要点速记：(1).** **如果仅为了实现简单事件通信，基于条件变量的设计会要求多余的互斥量，这会给相互关联的检测和反应任务带来约束，并要求反应任务校验事件确已发生。(2).** **使用标志位的设计可以避免上述问题，但这一设计基于轮询而非阻塞 (based on polling, not blocking)****。(3).** **条件变量和标志位可以一起使用，但这样的通信机制设计结果不甚自然 (somewhat stilted)****。(4).** **使用 std::promise** **型别对象和期值 (future)** **就可以回避这些问题，但是这个途径为了共享状态需要使用堆内存，而且仅限于一次性通信**。



std::promise 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104124174



## **40.** **对并发使用 std::atomic****，对特种内存使用 volatile(Use std::atomic for concurrency, volatile for special memory)**



```
int test_item_40()
{
	std::atomic<int> ai(0); // initialize ai to 0
	ai = 10; // atomically set ai to 10
	std::cout << ai << std::endl; // atomically read ai's value
	++ai; // atomically increment ai to 11
	--ai; // atomically decrement ai to 10
 
	volatile int x = 0;
	auto y = x; // read x
	y = x; // read x again(can't be optimized away,不可以被优化掉)
 
	x = 10; // write x(can't be optimized away)
	x = 20; // write x again
 
	std::atomic<int> x2;
	std::atomic<int> y2(x2.load()); // read x2
	y2.store(x2.load()); // read x2 again
 
	register int z2 = x2.load(); // read x2 into register
	std::atomic<int> y2_(z2); // init y2_ with register value
	y2_.store(z2); // store register value into y2_
 
	volatile std::atomic<int> vai; // operations on vai are atomic and can't be optimized away
 
	return 0;
}
```



std::atomic 模板的实例 (例如，std::atomic<int>, std::atomic<bool > 和 std::atomic<Widget*> 等)提供的操作可以保证被其它线程视为原子的。一旦构造了一个 std::atomic 型别对象，针对它的操作就好像这些操作处于受互斥量保护的临界区域内一样，但是实际上这些操作通常会使用特殊的机器指令来实现，这些指令比使用互斥量来得更加高效。



volatile 的用处就是告诉编译器，正在处理的是特种内存 (special memory)，它的意思是通知编译器” 不要对在此内存上的操作做任何优化”。可能最常见的特种内存是用于内存映射 I/O 的内存。这种内存的位置实际上是用于与外部设备 (例如，外部传感器、显示器、打印机和网络端口等) 通信，而非用于读取或写入常规内存(即 RAM)。编译器可以消除 std::atomic 型别上的冗余操作。std::atomic 对于并发程序设计有用，但不能用于访问特种内存。volatile 对于访问特种内存有用，但不能用于并发程序设计。由于 std::atomic 和 volatile 是用于不同目的，它们甚至可以一起使用。访问 std::atomic 型别对象通常比访问非 std::atomic 型别对象慢得多。



**要点速记：(1).std::atomic** **用于多线程访问的数据，且不用互斥量。它是编写并发软件的工具。(2).volatile** **用于读写操作不可以被优化掉的内存。它是在面对特种内存时使用的工具**。



std::atomic 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/73436710



volatile 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/104109696



## **41.** **针对可复制的形参，在移动成本低并且一定会被复制的前提下，才考虑将其按值传递 (Consider pass by value for copyable parameters that are cheap to move and always copied)**



```
class Widget41 {
public:
	 method 1: by-reference approaches
	//void addName(const std::string& newName) // 接受左值，对其实施拷贝
	//{
	//	names.push_back(newName);
	//}
 
	//void addName(std::string&& newName) // 接受右值，对其实施移动
	//{
	//	names.push_back(std::move(newName));
	//}
 
	 method2: by-reference approaches
	//template<typename T>
	//void addName(T&& newName) // 万能引用，接受左值也接受右值，对左值实施拷贝，对右值实施移动
	//{
	//	names.push_back(std::forward<T>(newName));
	//}
 
	// method3: by-value approaches
	void addName(std::string newName) // 即接受左值也接受右值，对右值实施移动
	{
		names.push_back(std::move(newName));
	}
 
private:
	std::vector<std::string> names;
};
 
int test_item_41()
{
	Widget41 w;
	std::string name("Bart");
	w.addName(name); // call addName with lvalue
	w.addName(name + "Jenne"); // call addName with rvalue
 
	return 0;
}
```



**要点速记：(1).** **对于可复制的、在移动成本低廉并且一定会被复制的形参而言，按值传递可能会和按引用传递具备相近的效率，并可能生成更少量的目标代码。(2).** **构造复制形参的成本可能比赋值复制形参高出很多。(3).** **按值传递肯定会导致切片问题 (slicing problem)****，所以基类型别特别不适用于按值传递**。



## **42.** **考虑置入而非插入 (Consider emplacement instead of insertion)**



```
int test_item_42()
{
	std::vector<std::string> vs;
	vs.push_back("xyzzy"); // 调用两次构造函数，一次析构函数
	vs.emplace_back("xyzzy"); // 调用一次构造函数，不涉及任何临时对象
 
	vs.emplace_back(50, 'x'); // insert std::string consisting of 50 'x' characters
 
	std::string queenOfDisco("Donna Summer");
	// 以下两条语句效果相同
	vs.push_back(queenOfDisco); // copy-construct queenOfDisco at end of vs
	vs.emplace_back(queenOfDisco); // copy-construct queenOfDisco at end of vs
 
	//std::regex r1 = nullptr; // 拷贝初始化(copy initialization),error, won't compile
	//std::regex r2(nullptr); // 直接初始化(direct initialization),can compile
 
	std::vector<std::regex> regexes;
	//regexes.emplace_back(nullptr); // 能编译，直接初始化允许使用接受指针的、带有explicit声明的std::regex构造函数
	//regexes.push_back(nullptr); // 不能编译，拷贝初始化禁止使用带指针的、explicit声明的std::regex构造函数
 
	return 0;
}
```



emplace_back 可用于任何支持 push_back 的标准容器。相似地，所有支持 push_front 的标准容器也支持 emplace_front。还有，任何支持插入 (insert) 操作 (即除了 std::forward_list 和 std::array 以外的所有标准容器) 都支持置入操作(emplace)。置入函数可避免临时对象的创建和析构，但插入函数就无法避免。即使在插入函数并不要求创建临时对象的情况下，也可以使用置入函数。在那种情况下，插入函数和置入函数本质上做的是同一件事。



如果下列情况都成立，那么置入将几乎肯定会比插入更高效：(1). 欲添加的值是以构造而非赋值方式加入容器。(2). 传递的实参型别与容器保存的型别不同。(3). 容器不太可能由于重复值而拒绝新值 (The container is unlikely to reject the new value as a duplicate)。



在决定是否选用置入函数时，还有其它两个问题值得操心：第一个和资源管理有关。第二个是它们与显式构造函数之间的交互 (interaction)。在使用置入函数时，要特别小心保证传递了正确的实参。



**要点速记：(1).** **从原理上说，置入函数 (emplacement function)** **应该有时比对应的插入函数高效，而且不应该有更低效的可能。(2).** **从实践上说，置入函数在以下几个前提成立时，极有可能会运行得更快：待添加的值是以构造而非赋值方式加入容器；传递的实参型别与容器保存的参数型别不同；容器不会因为重复值而拒绝待添加的值。(3).** **置入函数可能会执行类型转换，而插入函数会拒绝这些类型转换**。



emplace 更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/78670376



**GitHub**：https://github.com/fengbingchun/Messy_Test

