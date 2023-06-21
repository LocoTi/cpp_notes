# Effective STL

[Effective STL 50条有效使用STL的经验笔记_网络资源是无限的-CSDN博客_effective stl](https://blog.csdn.net/fengbingchun/article/details/103223914)

注：(1). 书中少许内容已不再适合现代 C++ 的开发，有些算法或函数在 C++11 中已不推荐使用，在下文涉及到的地方已标注。(2). 有些测试代码使用了 C++11 特性。



STL 表示 C++ 标准库中与迭代器一起工作的那部分，其中包括标准容器 (包含 string)、iostream 库的一部分、函数对象和各种算法。STL 主要组成部分包括：容器、迭代器、算法、函数对象。



根据迭代器所支持的操作，可以把迭代器分为五类：



(1). 输入迭代器 (input iterator) 是只读迭代器，在每个被遍历到的位置上只能被读取一次。



(2). 输出迭代器 (output iterator) 是只写迭代器，在每个被遍历到的位置上只能被写入一次。



输入和输出迭代器的模型分别是建立在针对输入和输出流 (例如文件) 的读写操作的基础上的。所以不难理解，输入和输出迭代器最常见的表现形式是 istream_iterator 和 ostream_iterator。



(3). 前向迭代器 (forward iterator) 兼具输入和输出迭代器的能力，但是它可以对同一个位置重复进行读和写。前向迭代器不支持 operator--，所以它只能向前移动。所有的标准 STL 容器都支持比前向迭代器功能更强大的迭代器。



(4). 双向迭代器 (bidirectional iterator) 很像前向迭代器，只是它们向后移动和向前移动同样容易。标准关联容器都提供了双向迭代器。list 也是如此。



(5). 随机访问迭代器 (random access iterator) 有双向迭代器的所有更能，而且，它还提供了”迭代器算术”，即在一步内向前或向后跳跃的能力。vector、string 和 deque 都提供了随机访问迭代器。指向数组内部的指针对于数组来说也是随机访问迭代器。



**所有重载了函数调用操作符 (****即 operator())** **的类都是一个函数子类 (functor class)****。从这些类创建的对象被称为函数对象 (function object)** **或函数子 (functor)**。在 STL 中，大多数使用函数对象的地方同样也可以使用实际的函数。



## **1.** **慎重选择容器类型**



STL 中有迭代器 (iterator)、算法(algorithm) 和函数对象(function object)。但是对于大多数 C++ 程序员来说，最值得注意的还是容器。容器比数组功能更强大、更灵活。它们可以动态增长(和缩减)，可以自己管理内存，可以记住自己包含了多少对象。它们限定了自己所支持的操作的复杂性。



标准 STL 序列容器：vector、string、deque、list、forward_list(C++11)、array(C++11)。



标准 STL 关联容器：set、multiset、map、multimap、unordered_set(C++11)、unordered_multiset(C++11)、unordered_map(C++11)、unordered_multimap(C++11)。



标准的非 STL 容器，包括：bitset(include <bitset>)、valarray(include <valarray>)。其它 STL 容器：stack(include <stack>)、queue(include <queue>) 和 priority_queue((include <queue>))。



vector<char> 可以作为 string 的替代。vector 作为标准关联容器的替代。有时 vector 在运行时间和空间上都要优于标准关联容器。



vector 是默认应使用的序列类型，当需要频繁地在序列中间做插入和删除操作时，应使用 list；当大部分插入和删除操作发生在序列的头部和尾部时， deque 是应考虑的数据结构。



**STL** **容器的一种分类方法**：连续内存容器 (contiguous memory container) 和基于节点的容器(node-based container)。



(1). 连续内存容器 (或称为基于数组的容器，array-based container) 把它的元素存放在一块或多块 (动态分配的) 内存中，每块内存中存有多个元素。当有新元素插入或已有的元素被删除时，同一内存块中的其它元素要向前或向后移动，以便为新元素让出空间，或者填充被删除元素所留下的空隙。这种移动影响到效率和异常安全性。标准的连续内存容器有 vector、string、deque。



(2). 基于节点的容器在每一个 (动态分配的) 内存块中只存放一个元素。容器中元素的插入或删除只影响到指向节点的指针，而不影响节点本身的内容，所以当有插入或删除操作时，元素的值不需要移动。表示链表的容器，比如 list、forward_list，是基于节点的；所有的标准关联容器也是如此(通常的实现方式是平衡树)。标准的哈希容器使用不同的基于节点的实现。



选择容器时最重要的一些问题：



(1). 你是否需要在容器的任意位置插入新元素？如果需要，就选择序列容器；关联容器是不行的。



(2). 你是否关心容器中的元素是如何排序的？如果不关心，则哈希容器是一个可行的选择方案；否则，你要避免哈希容器。



(3). 你选择的容器必须是标准 C++ 的一部分吗？如果必须是，就排除了 slist 和 rope。



(4). 你需要哪种类型的迭代器？如果它们必须是随机访问迭代器，则对容器的选择就被限定为 vector、deque 和 string。或许你也可以考虑 rope。如果要求使用双向迭代器，那么你必须避免 slist 以及哈希容器的一个常见实现。



(5). 当发生元素的插入或删除操作时，避免移动容器中原来的元素是否很重要？如果是，就要避免连续内存的容器。



(6). 容器中数据的布局是否需要和 C 兼容？如果需要兼容，就只能选择 vector。



(7). 元素的查找速度是否是关键的考虑因素？如果是，就要考虑哈希容器、排序的 vector 和标准关联容器 ---- 或许这就是优先顺序。



(8). 如果容器内部使用了引用计数技术 (reference counting)，你是否介意？如果是，就要避免使用 string，因为许多 string 的实现都使用了引用计数。rope 也需要避免，因为权威的 rope 实现是基于引用计数的。当然，你需要某种表示字符串的方法，这时你可以考虑 vector<char>。



(9). 对插入和删除操作，你需要事务语义 (transactional semantics) 吗？也就是说，在插入和删除操作失败时，你需要回滚的能力吗？如果需要，你就要使用基于节点的容器。如果对多个元素的插入操作 (即针对一个区间的形式) 需要事务语义，则你需要选择 list，因为在标准容器中，只有 list 对多个元素的插入操作提供了事务语义。对那些希望编写异常安全 (exception-safe) 代码的程序员，事务语义显得尤为重要。(使用连续内存的容器也可以获得事务语义，但是要付出性能上的代价，而且代码也显得不那么直截了当。)



(10). 你需要使迭代器、指针和引用变为无效的次数最少吗？如果是这样，就要使用基于节点的容器，因为对这类容器的插入和删除操作从来不会使迭代器、指针和引用变得无效 (除非它们指向了一个你正在删除的元素)。而针对连续内存容器的插入和删除操作一般会使指向该容器的迭代器、指针和引用变得无效。



(11). 如果序列容器的迭代器是随机访问类型，而且只要没有删除操作发生，且插入操作只发生在容器的末尾，则指向数据的指针和引用就不会变为无效，这样的容器是否对你有帮助？这是非常特殊的情形，但如果你面对的情形正是如此，则 deque 是你所希望的容器。(当插入操作仅在容器末尾发生时，deque 的迭代器有可能会变为无效。deque 是唯一的、迭代器可能会变为无效而指针和引用不会变为无效的 STL 标准容器。)



## **2.** **不要试图编写独立于容器类型的代码**



```
int test_item_2()
{
	// 对容器类型和其迭代器类型使用类型定义(typedef)
	typedef std::vector<int> WidgetContainer;
	typedef WidgetContainer::iterator WCIterator;
	WidgetContainer cw;
	int bestWidget;
 
	WCIterator i = std::find(cw.begin(), cw.end(), bestWidget);
 
	return 0;
}
```



**STL** **是以泛化 (generalization)** **原则为基础的**：数组被泛化为” 以其包含的对象的类型为参数” 的容器，函数被泛化为” 以其使用的迭代器的类型为参数” 的算法，指针被泛化为” 以其指向的对象的类型为参数” 的迭代器。



容器类型被泛化为序列和关联容器，类似的容器被赋予相似的功能。标准的连续内存容器提供了随机访问迭代器，而标准的基于节点的容器提供了双向迭代器。序列容器支持 push_front 和 / 或 push _back 操作，而关联容器则不然。关联容器提供了对数时间的 lower_bound、upper_bound 和 equal_range 成员函数，但序列容器却没有提供。



试图编写独立于容器的代码 (container independent code)，这类泛化，尽管出发点是好的，却几乎总是误入歧途。试图编写对序列容器和关联容器都适用的代码几乎是毫无意义的。很多成员函数仅当其容器为某一种类型时才存在。不同的容器是不同的，它们有非常明显的优缺点。它们并不是被设计来交换适用的。



类型定义只不过是其它类型的别名，所以它带来的封装纯粹是词法 (lexical) 上的。类型定义并不能阻止一个客户去做 (或依赖) 它们原本无法做到 (或依赖) 的事情。



要想减少在替换容器类型时所需要修改的代码，你可以把容器隐藏到一个类中，并尽量减少那些通过类接口 (而使外部) 可见的、与容器相关的信息。



## **3.** **确保容器中的对象拷贝正确而高效**



```
class Widget {};
class SpecialWidget : public Widget {};
 
int test_item_3()
{
	std::vector<Widget> vw;
	SpecialWidget sw;
	vw.push_back(sw); // sw作为基类对象被拷贝进vw中，它的派生类特有部分在拷贝时被丢掉了
 
	return 0;
}
```



当 (通过如 insert 或 push_back 之类的操作) 向容器中加入对象时，存入容器的是你所指定的对象的拷贝。当 (通过如 front 或 back 之类的操作) 从容器中取出一个对象时，你所得到的是容器中所保存的对象的拷贝。进去的是拷贝，出来的也是拷贝(copy in, copy out)。这就是 STL 的工作方式。



一旦一个对象被保存到容器中，它经常会进一步被拷贝。当对 vector、string 或 deque 进行元素的插入或删除操作时，现有元素的位置通常会被移动 (拷贝)。如果你使用下列任何操作 ---- 排序算法，next_permutation 或 previous_permutation, remove、unique 或类似的操作，rotate 或 reverse, 等等 ---- 那么对象将会被移动 (拷贝)。没错，拷贝对象是 STL 的工作方式。



利用一个对象的拷贝成员函数就可以很方便地拷贝该对象，特别是对象的拷贝构造函数 (copy constructor) 和拷贝赋值操作符(copy assignment operator)。



在存在继承关系的情况下，拷贝动作会导致剥离 (slicing)。也就是说，如果你创建了一个存放基类对象的容器，却向其中插入派生类的对象，那么在派生类对象(通过基类的拷贝构造函数) 被拷贝进容器时，它所特有的部分 (即派生类中的信息) 将会丢失。**”** **剥离”** **问题意味着向基类对象的容器中插入派生类对象几乎总是错误的**。**使拷贝动作高效、正确，并防止剥离问题发生的一个简单办法是使容器包含指针而不是对象**。



## **4.** **调用 empty** **而不是检查 size()** **是否为 0**



empty 对所有的标准容器都是常数时间操作，而对一些 list 实现，size 耗费线性时间。



## **5.** **区间成员函数优先于与之对应的单元素成员函数**



```
class Widget5 {};
 
int test_item_5()
{
	std::vector<Widget5> v1, v2;
	v1.assign(v2.begin() + v2.size() / 2, v2.end()); // 推荐
 
	v1.clear();
	for (std::vector<Widget5>::const_iterator ci = v2.begin() + v2.size() / 2; ci != v2.end(); ++ci) // 不推荐
		v1.push_back(*ci);
 
	v1.clear();
	std::copy(v2.begin() + v2.size() / 2, v2.end(), std::back_inserter(v1)); // 效率不如assign
 
	v1.clear();
	v1.insert(v1.end(), v2.begin() + v2.size() / 2, v2.end()); // 对copy的调用可以被替换为利用区间的insert版本
 
	const int numValues = 100;
	int data[numValues];
 
	std::vector<int> v;
	v.insert(v.begin(), data, data + numValues); // 推荐，使用区间成员函数insert
 
	std::vector<int>::iterator insertLoc(v.begin());
	for (int i = 0; i < numValues; ++i) {
		insertLoc = v.insert(insertLoc, data[i]); // 不推荐，使用单元素成员函数
		++insertLoc;
	}
 
	return 0;
}
```



区间成员函数是指这样的一类成员函数，它们像 STL 算法一样，使用两个迭代器参数来确定该成员操作所执行的区间。如果不使用区间成员函数就得写一个显示的循环。



所有通过利用插入迭代器 (insert iterator) 的方式 (即利用 inserter、back_inserter 或 front_inserter) 来限定目标区间的 copy 调用，其实都可以 (也应该) 被替换为对区间成员函数的调用。



优先选择区间成员函数而不是其对应的单元素成员函数有三条充分的理由：区间成员函数写起来更容易，更能清楚地表达你的意图，而且它们表现出了更高的效率。



## **6.** **当心 C++** **编译器最烦人的分析机制**



```
// 注意：围绕参数名的括号(比如对f2中d)与独立的括号的区别：围绕参数名的括号被忽略，而独立的括号则表明参数
// 列表的存在：它们说明存在一个函数指针参数
int f1(double d); // 声明了一个带double参数并返回int的函数
int f2(double(d)); // 同上，d两边的括号被忽略,可以给参数名加上圆括号
int f3(double); // 同上，参数名被忽略
 
int g1(double(*pf)()); // 参数是一个指向不带任何参数的函数的指针，该函数返回double值；g1以指向函数的指针为参数
int g2(double pf()); // 同上，pf为隐式指针
int g3(double()); // 同上，省去参数名
 
int test_item_6()
{
	// 把一个存有整数(int)的文件ints.dat拷贝到一个list中
	std::ifstream dataFile("ints.dat");
	std::list<int> data1(std::istream_iterator<int>(dataFile), std::istream_iterator<int>()); // 小心，结果不会是你所想象的那样
 
	std::list<int> data2((std::istream_iterator<int>(dataFile)), std::istream_iterator<int>()); // 正确，注意list构造函数的第一个参数两边的括号
 
	std::istream_iterator<int> dataBegin(dataFile);
	std::istream_iterator<int> dataEnd;
	std::list<int> data3(dataBegin, dataEnd); // 正确
 
	return 0;
}
```



使用命名的迭代器对象与通常的 STL 程序风格相违背，但你或许觉得为了使代码对所有编译器都没有二义性，并且使维护代码的人理解起来更容易，这一代价是值得的。



## **7.** **如果容器中包含了通过 new** **操作创建的指针，切记在容器对象析构前将指针 delete** **掉**



```
class Widget7 {};
 
struct DeleteObject {
	template<typename T>
	void operator()(const T* ptr) const
	{
		delete ptr;
	}
};
 
int test_item_7()
{
	const int num = 5;
 
	std::vector<Widget7*> vwp1, vwp2;
	for (int i = 0; i < num; ++i) {
		vwp1.push_back(new Widget7); // 如果在后面自己不delete，使用vwp在这里发生了Widget7的泄露
		vwp2.push_back(new Widget7);
	}
 
	for (std::vector<Widget7*>::iterator i = vwp1.begin(); i != vwp1.end(); ++i) {
		delete *i; // 能行，但不是异常安全的
	}
 
	for_each(vwp2.begin(), vwp2.end(), DeleteObject()); // 正确，类型安全，但仍不是异常安全的
 
	typedef std::shared_ptr<Widget7> SPW; // SPW"指向Widget7的shared_ptr"
	std::vector<SPW> vwp3;
	for (int i = 0; i < num; ++i) {
		vwp3.push_back(SPW(new Widget7)); // 从Widget7创建SPW,然后对它进行一次push_back使用vwp3,这里不会有Widget7泄露，即使有异常被抛出
	}
 
	return 0;
}
```



STL 容器很智能，但没有智能到知道是否该删除自己所包含的指针的程度。当你使用指针的容器，而其中的指针应该被删除时，为了避免资源泄漏，你必须或者用引用计数形式的智能指针对象 (比如 std::shared_ptr) 代替指针，或者当容器被析构时手工删除其中的每个指针。



std::shared_ptr 的使用参考：https://blog.csdn.net/fengbingchun/article/details/52202007



## **8.** **切勿创建包含 auto_ptr** **的容器对象**



auto_ptr 的容器 (简称 COAP) 是被禁止的。当你拷贝一个 auto_ptr 时，它所指向的对象的所有权被移交到拷入的 auto_ptr 上，而它自身被置为 NULL。如果你的目标是包含智能指针的容器，这并不意味着你要倒霉，包含智能指针的容器是没有问题的。



## **9.** **慎重选择删除元素的方法**



```
bool badValue(int) { return true; } // 返回x是否为"坏值"
 
int test_item_9()
{
	// 删除c中所有值为1963的元素
	std::vector<int> c1;
	c1.erase(std::remove(c1.begin(), c1.end(), 1963), c1.end()); // 当c1是vector, string或deque时，erase-remove习惯用法是删除特定值的元素的最好办法
 
	std::list<int> c2;
	c2.remove(1963); // 当c2是list时，remove成员函数是删除特定值的元素的最好办法
 
	std::set<int> c3;
	c3.erase(1963); // 当c3是标准关联容器时，erase成员函数是删除特定值元素的最好办法
 
	// 删除判别式(predicate)返回true的每一个对象
	c1.erase(std::remove_if(c1.begin(), c1.end(), badValue), c1.end()); // 当c1是vector, string或deque时，这是删除使badValue返回true的对象的最好办法
 
	c2.remove_if(badValue); // 当c2是list时，这是删除使badValue返回true的对象的最好办法
 
	for (std::set<int>::iterator i = c3.begin(); i != c3.end();) {
		if (badValue(*i)) c3.erase(i++); // 对坏值，把当前的i传给erase，递增i是副作用
		else ++i;                        // 对好值，则简单的递增i
	}
 
	// 每次元素被删除时，都向一个日志(log)文件中写一条信息
	std::ofstream logFile;
	for (std::set<int>::iterator i = c3.begin(); i != c3.end();) {
		if (badValue(*i)) {
			logFile << "Erasing " << *i << '\n'; // 写日志文件
			c3.erase(i++); // 对坏值，把当前的i传给erase，递增i是副作用
		}
		else ++i;              // 对好值，则简单第递增i
	}
 
	for (std::vector<int>::iterator i = c1.begin(); i != c1.end();) {
		if (badValue(*i)) {
			logFile << "Erasing " << *i << '\n';
			i = c1.erase(i); // 把erase的返回值赋给i，使i的值保持有效
		}
		else ++i;
	}
 
	return 0;
}
```



(1). 要删除容器中有特定值的所有对象：如果容器是 vector, string 或 deque，则使用 erase-remove 习惯用法；如果容器是 list，则使用 list::remove；如果容器是一个标准关联容器，则使用它的 erase 成员函数。



(2). 要删除容器中满足特定判别式 (条件) 的所有对象：如果容器是 vector, string 或 deque，则使用 erase-remove_if 习惯用法；如果容器是 list，则使用 list::remove_if；如果容器是一个标准关联容器，则使用 remove_copy_if 和 swap，或者写一个循环来遍历容器中的元素，记住当把迭代器传给 erase 时，要对它进行后缀递增。



(3). 要在循环内做某些 (除了删除对象之外的) 操作：如果容器是一个标准序列容器，则写一个循环来遍历容器中的元素，记住每次调用 erase 时，要用它的返回值更新迭代器；如果容器是一个标准关联容器，则写一个循环来遍历容器中的元素，记住当把迭代器传给 erase 时，要对迭代器做后缀递增。



## **10.** **了解分配子 (allocator)** **的约定和限制**



编写自定义的分配子，需要注意：(1). 你的分配子是一个模板，模板参数 T 代表你为它分配内存的对象的类型。(2). 提供类型定义 pointer 和 reference，但是始终让 pointer 为 T*，reference 为 T&。(3). 千万别让你的分配子拥有随对象而不同的状态 (per-object state)。通常，分配子不应该有非静态的数据成员。(4). 记住，传给分配子的 allocate 成员函数的是那些要求内存的对象的个数，而不是所需的字节数。同时要记住，这些函数返回 T * 指针 (通过 pointer 类型定义)，即使尚未有 T 对象被构造出来。(5). 一定要提供嵌套的 rebind 模板，因为标准容器依赖该模板。



std::allocator 的使用参考：https://blog.csdn.net/fengbingchun/article/details/78943527



## **11.** **理解自定义分配子的合理用法**



```
void* mallocShared(size_t bytesNeeded)
{
	return malloc(bytesNeeded);
}
 
void freeShared(void* ptr)
{
	free(ptr);
}
 
template<typename T>
class SharedMemoryAllocator { // 把STL容器的内容放到共享内存(即由mallocShared生成的)中去
public:
	typedef T* pointer; // pointer是个类型定义，它实际上总是T*
	typedef size_t size_type; // 通常情况下,size_type是size_t的一个类型定义
	typedef T value_type;
 
	pointer allocate(size_type numObjects, const void* localityHint = 0)
	{
		return static_cast<pointer>(mallocShared(numObjects * sizeof(T)));
	}
 
	void deallocate(pointer ptrToMemory, size_type numObjects)
	{
		freeShared(ptrToMemory);
	}
 
	template<typename U>
	struct rebind {
		typedef std::allocator<U> other;
	};
};
 
int test_item_11()
{
	typedef std::vector<double, SharedMemoryAllocator<double>> SharedDoubleVec;
	// v所分配的用来容纳其元素的内存将来自共享内存
	// 而v自己----包括它所有的数据成员----几乎肯定不会位于共享内存中，v只是普通的基于栈(stack)的对象，所以，像所
	// 有基于栈的对象一样，它将会被运行时系统放在任意可能的位置上。这个位置几乎肯定不是共享内存
	SharedDoubleVec v; // 创建一个vector,其元素位于共享内存中
 
	// 为了把v的内容和v自身都放到共享内存中，需要这样做
	void* pVectorMemory = mallocShared(sizeof(SharedDoubleVec)); // 为SharedDoubleVec对象分配足够的内存
	SharedDoubleVec* pv = new (pVectorMemory)SharedDoubleVec; // 使用"placement new"在内存中创建一个SharedDoubleVec对象
	// ... // 使用对象(通过pv)
	pv->~SharedDoubleVec(); // 析构共享内存中的对象
	freeShared(pVectorMemory); // 释放最初分配的那一块共享内存
 
	return 0;
}
```



遵守同一类型的分配子必须是等价的这一限制要求。



## **12.** **切勿对 STL** **容器的线程安全性有不切实际的依赖**



当涉及到 STL 容器和线程安全性时，你可以指望一个 STL 库允许多个线程同时读一个容器，以及多个线程对不同的容器做写入操作。你不能指望 STL 库会把你从手工同步控制中解脱出来，而且你不能依赖于任何线程支持。



## **13. vector** **和 string** **优先于动态分配的数组**



std::vector 的使用参考：https://blog.csdn.net/fengbingchun/article/details/51510916



std::string 的使用参考：https://blog.csdn.net/fengbingchun/article/details/62417284



**许多 string** **实现在背后使用了引用计数技术**，这种策略可以消除不必要的内存分配和不必要的字符拷贝，从而可以提供很多应用程序的效率。如果你在多线程环境下使用了引用计数的 string，那么注意一下因支持线程安全而导致的性能问题。**vector** **的实现不允许使用引用计数**，所以不会发生隐藏的多线程性能问题。



## **14.** **使用 reserve** **来避免不必要的重新分配**



```
int test_item_14()
{
	std::vector<int> v;
	v.reserve(1000); // 如果不使用reserve,下面的循环在进行过程中将导致2到10次重新分配;加上reserve，则在循环过程中,将不会再发生重新分配
	for (int i = 1; i <= 1000; ++i) v.push_back(i);
 
	return 0;
}
```



对于 vector 和 string，增长过程是这样来实现的：每当需要更多空间时，就调用与 realloc 类似的操作。这一类似于 realloc 的操作分为四部分：(1). 分配一块大小为当前容量的某个倍数的新内存。在大多数实现中，vector 和 string 的容量每次以 2 的倍数增长，即，每当容器需要扩张时，它们的容量即加倍。(2). 把容器的所有元素从旧的内存拷贝到新的内存中。(3). 析构掉就内存中的对象。(4). 释放旧内存。



reserve 成员函数能使你把重新分配的次数减少到最低限度，从而避免了重新分配和指针 / 迭代器 / 引用失效带来的开销。避免重新分配的关键在于，尽早地使用 reserve，把容器的容量设为足够大的值，最好是在容器刚被构造出来之后就使用 reserve。



大小 (size) 和容量 (capacity) 之间的关系使我们能够预知什么时候插入操作会导致 vector 或 string 进行重新分配的动作，进而又使我们有可能预知什么时候一个插入操作会使容器中的迭代器、指针和引用失效。



通常有两种方式来使用 reserve 以避免不必要的重新分配。第一种方式是，若能确切知道或大致预计容器中最终会有多少元素，则此时可以使用 reserve。第二种方式是，先预留足够大的空间 (根据你的需要而定)，然后，当把所有数据都加入以后，再去除多余的容量。



## **15.** **注意 string** **实现的多样性**



```
int test_item_15()
{
	fprintf(stdout, "string size: %d, char* size: %d\n", sizeof(std::string), sizeof(char*));
 
	return 0;
}
```



(1).string 的值可能会被引用计数，也可能不会。很多实现在默认情况下会使用引用计数，但它们通常提供了关闭默认选择的方法，往往是通过预处理宏来做到这一点。(2).string 对象大小的范围可以是一个 char * 指针大小的 1 倍到 7 倍。(3). 创建一个新的字符串值可能需要零次、一次或两次动态分配内存。(4).string 对象可能共享，也可能不共享其大小和容量信息。(5).string 可能支持，也可能不支持针对单个对象的分配子。(6). 不同的实现对字符内存的最小分配单位有不同的策略。



## **16.** **了解如何把 vector** **和 string** **数据传给旧的 API**



```
void doSomething(const int* pInts, size_t numInts) {}
void doSomething(const char* pString) {}
size_t fillArray(double* pArray, size_t arraySize) { return (arraySize - 2); }
size_t fillString(char* pArray, size_t arraySize) { return (arraySize - 2); }
 
int test_item_16()
{
	// vector/string --> C API
	std::vector<int> v{ 1, 2 };
	if (!v.empty()) {
		doSomething(&v[0], v.size());
		doSomething(v.data(), v.size()); // C++11
		// doSomething(v.begin(), v.size()); // 错误的用法
		doSomething(&*v.begin(), v.size()); // 可以，但不易于理解
	}
 
	std::string s("xxx");
	doSomething(s.c_str());
 
	// C API 初始化vector
	const int maxNumDoubles = 100;
	std::vector<double> vd(maxNumDoubles); // 创建大小为maxNumDoubles的vector
	vd.resize(fillArray(&vd[0], vd.size())); // 使用fillArray向vd中写入数据，然后把vd的大小改为fillArray所写入的元素的个数
 
	// C API 初始化string
	const int maxNumChars = 100;
	std::vector<char> vc(maxNumChars); // 创建大小为maxNumChars的vector
	size_t charsWritten = fillString(&vc[0], vc.size()); // 使用fillString向vc中写入数据
	std::string s2(vc.cbegin(), vc.cbegin() + charsWritten); // 通过区间构造函数，把数据从vc拷贝到s2中
 
	// 先让C API把数据写入到一个vector中，然后把数据拷贝到期望最终写入的STL容器中，这一思想总是可行的
	std::deque<double> d(vd.cbegin(), vd.cend()); // 把数据拷贝到deque中
	std::list<double> l(vd.cbegin(), vd.cend()); // 把数据拷贝到list中
	std::set<double> s22(vd.cbegin(), vd.cend()); // 把数据拷贝到set中
 
	// 除了vector和string以外的其它STL容器也能把它们的数据传递给C API，你只需要把每个容器的元素拷贝到一个vector中，然后传给该API
	std::set<int> intSet; // 存储要传给API的数据的set
	std::vector<int> v2(intSet.cbegin(), intSet.cend()); // 把set的数据拷贝到vector
	if (!v2.empty()) doSomething(&v2[0], v2.size()); // 把数据传给API
 
	return 0;
}
```



C++ 标准要求 vector 中的元素存储在连续的内存中，就像数组一样。string 中的数据不一定存储在连续的内存中，而且 string 的内部表示不一定是以空字符结尾的。



## **17.** **使用”swap** **技巧”** **除去多余的容量**



```
class Contestant {};
 
int test_item_17()
{
	// 从contestants矢量中除去多余的容量
	std::vector<Contestant> contestants;
	// ... // 让contestants变大，然后删除它的大部分元素
	// vector<Contestant>(contestants)创建一个临时矢量，vector的拷贝构造函数只为所拷贝的元素分配所需要的内存
	std::vector<Contestant>(contestants).swap(contestants);
 
	contestants.shrink_to_fit(); // C++11
 
	std::string s;
	// ... // 让s变大，然后删除它的大部分字符
	std::string(s).swap(s);
 
	s.shrink_to_fit(); // C++11
 
	std::vector<Contestant>().swap(contestants); // 清除contestants并把它的容量变为最小
 
	std::string().swap(s); // 清除s并把它的容量变为最小
 
	return 0;
}
```



对 vector 或 string 进行 shrink-to-fit 操作时，考虑”swap” 技巧。**C++11** **中增加了 shrink_to_fit** **成员函数**。



swap 技巧的一种变化形式可以用来清除一个容器，并使其容量变为该实现下的最下值。



在做 swap 的时候，不仅两个容器的内容被交换，同时它们的迭代器、指针和引用也将被交换 (string 除外)。在 swap 发生后，原先指向某容器中元素的迭代器、指针和引用依然有效，并指向同样的元素 ---- 但是，这些元素已经在另一个容器中了。



## **18.** **避免使用 vector<bool>**



```
int test_item_18()
{
	std::vector<bool> v;
	// error: cannot convert 'std::vector<bool>::reference* {aka std::_Bit_reference}' to 'bool*' in initialization
	//bool* pb = &v[0]; // 不能被编译，原因：vector<bool>是一个假的容器，它并不真的储存bool，相反，为了节省空间，它储存的是bool的紧凑表示
 
	return 0;
}
```



作为一个 STL 容器，vector<bool> 只有两点不对。首先，它不是一个 STL 容器；其次，它并不存储 bool。除此以外，一切正常。



在一个典型的实现中，储存在”vector”中的每个”bool”仅占一个二进制位，一个 8 位的字节可容纳 8 个”bool”。在内部，vector<bool> 使用了与位域 (bit field) 一样的思想，来表示它所存储的那些 bool，实际上它只是假装存储了这些 bool。



位域与 bool 相似，它只能表示两个可能的值，但是在 bool 和看似 bool 的位域之间有一个很重要的区别：你可以创建一个指向 bool 的指针，而指向单个位的指针则是不允许的。指向单个位的引用也是被禁止的。



当你需要 vector<bool> 时，标准库提供了两种选择，可以满足绝大多数情况下的需求。第一种是 deque<bool>。deque 几乎提供了 vector 所提供的一切 (没有 reserve 和 capacity)，但 deque<bool > 是一个 STL 容器，而且它确实存储 bool。当然 deque 中元素的内存不是连续的，所以你不能把 deque<bool > 中的数据传递给一个期望 bool 数组的 C API。



第二种可以替代 vector<bool> 的选择是 bitset。bitset 不是 STL 容器，但它是标准 C++ 库的一部分。与 STL 容器不同的是，它的大小 (即元素的个数) 在编译时就确定了，所以它不支持插入和删除元素。



## **19.** **理解相等 (equality)** **和等价 (equivalence)** **的区别**



std::set 的使用参考：https://blog.csdn.net/fengbingchun/article/details/63268962



std::map 的使用参考：https://blog.csdn.net/fengbingchun/article/details/52074286



相等的概念是基于 operator== 的。等价关系是以” 在已排序的区间中对象值的相对顺序” 为基础的。标准关联容器是基于等价而不是相等。



标准关联容器总是保持排列顺序的，所以每个容器必须有一个比较函数 (默认为 less) 来决定保持怎样的顺序。等价的定义正是通过该比较函数而确定的，因此，标准关联容器的使用者要为所使用的每个容器指定一个比较函数(用来决定如何排序)。如果该关联容器使用相等来决定两个对象是否有相同的值，那么每个关联容器除了用于排序的比较函数外，还需要另一个比较函数来决定两个值是否相等。(默认情况下，该比较函数应该是 equal_to，但 equal_to 从来没有被用作 STL 的默认比较函数。当 STL 中需要相等判断时，一般的惯例是直接调用 operator==)。



C++ 标准对于等价的值 (对 multiset) 或键 (对 multimap) 的相对顺序没有什么限制。



## **20.** **为包含指针的关联容器指定比较类型**



```
struct StringPtrLess : public std::binary_function<const std::string*, const std::string*, bool> { // std::binary_function在C++11中已被废弃
	bool operator()(const std::string* ps1, const std::string* ps2) const
	{
		return *ps1 < *ps2;
	}
};
 
struct DereferenceLess {
	template<typename PtrType>
	bool operator()(PtrType pT1, PtrType pT2) const
	{
		return *pT1 < *pT2;
	}
};
 
int test_item_20()
{
	//std::set<std::string*> ssp; // std::set<std::string*> <==> std::set<std::string*, std::less<std::string*>>, 得不到预期的结果
	//std::set<std::string*, StringPtrLess> ssp;
	std::set<std::string*, DereferenceLess> ssp; // 与std::set<std::string*, StringPtrLess> ssp;的行为相同
	ssp.insert(new std::string("Anteater"));
	ssp.insert(new std::string("Wombat"));
	ssp.insert(new std::string("Lemur"));
	ssp.insert(new std::string("Penguin"));
 
	for (auto it = ssp.cbegin(); it != ssp.cend(); ++it) {
		fprintf(stdout, "%s\n", (**it).c_str());
	}
 
	return 0;
}
```



每当你要创建包含指针的关联容器时，一定要记住，容器将会按照指针的值进行排序。绝大多数情况下，这不会是你所希望的，所以你几乎肯定要创建自己的函数子类作为该容器的比较类型 (comparison type)。



如果你有一个包含智能指针或迭代器的容器，那么你也要考虑为它指定一个比较类型。对指针的解决方案同样也适用于那些类似指针的对象。就像 DereferenceLess 适合作为包含 T * 的关联容器的比较类型一样，对于容器中包含了指向 T 对象的迭代器或智能指针的情形，DereferenceLess 也同样可用作比较类型。



## **21.** **总是让比较函数在等值情况下返回 false**



比较函数的返回值表明的是按照该函数定义的排列顺序，一个值是否在另一个之前。相等的值从来不会有前后顺序关系，所以，对于相等的值，比较函数应当始终返回 false。对 set 和 map 确实是这样，因为这些容器不能包含重复的值。对 multiset 和 multimap 也是这样。从技术上来说，用于对关联容器排序的比较函数必须为它们所比较的对象定义一个” 严格的弱序化”(strict weak ordering)。(对于传递给像 sort 这类算法的比较函数也有同样的限制。) 任何一个定义了” 严格的弱序化” 的函数必须对相同值的两个拷贝返回 false。



## **22.** **切勿直接修改 set** **或 multiset** **中的键**



```
int test_item_22()
{
	std::map<int, std::string> m{ { 0, "xxx" } };
	//m.begin()->first = 10; // build error, map的键不能修改
 
	std::multimap<int, std::string> mm{ { 1, "yyy" } };
	//mm.begin()->first = 10; // build error, multimap的键同样不能修改
 
	std::set<int> s{ 1, 2, 3 };
	//*(s.begin()) = 10; // build error, set的键不能修改
	const_cast<int&>(*s.begin()) = 10; // 强制类型转换
 
	std::vector<int> v{ 1, 2, 3 };
	*v.begin() = 10;
 
	return 0;
}
```



像所有的标准关联容器一样，set 和 multiset 按照一定的顺序来存放自己的元素，而这些容器的正确行为也是建立在其元素保持有序的基础之上的。如果你把关联容器中的一个元素的值改变了 (比如把 10 改为 1000)，那么，新的值可能不在正确的位置上，这将会打破容器的有序性。



对于 map 和 multimap 尤其简单，因为如果有程序试图改变这些容器中的键，它将不能通过编译。这是因为，对于一个 map<K, V> 或 multimap<K, V > 类型的对象，其中的元素类型是 pair<const K, V>。因为键的类型是 const K，所以它不能被修改。(如果利用 const_cast，你或许可以修改它。)



对于 set<T> 或 multiset<T > 类型的对象，容器中元素的类型是 T，而不是 const T。因此，只要你愿意，你随时可以改变 set 或 multiset 中的元素，而无需任何强制类型转换。注：不通过强制类型转换并不能改变 set 或 multiset 中的元素。



## **23.** **考虑用排序的 vector** **替代关联容器**



对于许多应用，哈希容器可能提供常数时间的查找能力优于 set、multiset、map 和 multimap 的确定的对数时间查找能力。



标准关联容器通常被实现为平衡的二叉查找树。



在排序的 vector 中存储数据可能比在标准关联容器中存储同样的数据要耗费更少的内存，而考虑到页面错误的因素，通过二分搜索法来查找一个排序的 vector 可能比查找一个标准关联容器要更快一些。当然，对于排序的 vector，最不利的地方在于它必须保持有序，当一个新的元素被插入时，新元素之后的所有元素都必须向后移动一个元素的位置，尤其是当 vector 必须重新分配自己的内存时。与此类似，如果一个元素被从 vector 中删除了，则在它之后的所有元素也都要向前移动。插入和删除操作对于 vector 来说是昂贵的，但对于关联容器却是廉价的。这就是为什么只有当你知道” 对数据结构的使用方式是：**查找操作几乎从不跟插入和删除操作混在一起****”** **时，再考虑使用排序的 vector** **而不是关联容器才是合理的**。否则，使用排序的 vector 而不是标准关联容器几乎肯定是在浪费时间。



## **24.** **当效率至关重要时，请在 map::operator[]** **与 map::insert** **之间谨慎做出选择**



```
// 提供高效的添加和更新功能
template<typename MapType, typename KeyArgType, typename ValueArgType>
typename MapType::iterator efficientAddOrUpdate(MapType& m, const KeyArgType& k, const ValueArgType& v)
{
	typename MapType::iterator lb = m.lower_bound(k); // 确定k在什么位置或应在什么位置
 
	if (lb != m.end() && !(m.key_comp()(k, lb->first))) { // 如果lb指向的pair的健与k等价,
		lb->second = v;				      // 则更新pair的值并返回指向该pair的迭代器
		return lb;
	}
	else {
		typedef typename MapType::value_type MVT;
		return m.insert(lb, MVT(k, v)); // 把pair<k,v>添加到m中，并返回一个指向该新元素的迭代器
	}
}
 
int test_item_24()
{
	std::map<int, std::string> m;
	m[1] = "xxx"; // m[1]是m.operator[](1)的缩写形式
	m.operator[](1) = "xxx";
 
	// m[1] = "xxx"; 在功能上等同于
	typedef std::map<int, std::string> IntStrMap;
	std::pair<IntStrMap::iterator, bool> result = m.insert(IntStrMap::value_type(1, std::string()));
	result.first->second = "xxx";
	// 以上方式性能降低的原因：先默认构造了一个string，然后立刻赋给它新的值。如果"直接使用我们所需要的值构造一个
	// string"比"先默认构造一个string再赋值"效率更高，那么，我们最好把对operator[]的使用(包括与之相伴的构造和
	// 赋值)换成对insert的直接调用
	m.insert(IntStrMap::value_type(1, "xxx")); // 它通常会节省三个函数调用：一个用于创建默认构造的临时string对象，
	// 一个用于析构该临时对象，另一个是调用string的赋值操作符
 
	efficientAddOrUpdate(m, 2, "yyy");
 
	return 0;
}
```



map 的 operator[] 函数与众不同。它与 vector、deque 和 string 的 operator[] 函数无关，与用于数组的内置 operator[] 也没有关系。相反，map::operator[] 的设计目的是为了提供” 添加和更新”(add or update) 的功能。map::operator[] 返回一个引用。



对效率的考虑使我们得出结论：当向映射表中添加元素时，要优先选用 insert，而不是 operator[]；而从效率和美学的观点考虑，结论是：当更新已经在映射表中的元素的值时，要优先选择 operator[]。



## **25.** **熟悉非标准的哈希容器**



C++11 中新增了四种关联容器，使用哈希函数组织的，无序的，即 unordered_map、unordered_multimap、unordered_set、unordered_multiset。



std::unordered_map 的使用参考：https://blog.csdn.net/fengbingchun/article/details/52235026



## **26. iterator** **优先于 const_iterator****、reverse_iterator** **以及 const_reverse_iterator**



STL 中的所有标准容器都提供了 4 中迭代器类型。对容器类 container<T> 而言，iterator 类型的功效相当于 T*，而 const_iterator 则相当于 const T*。对一个 iterator 或者 const_iterator 进行递增则可以移动到容器中的下一个元素，通过这种方式可以从容器的头部一直遍历到尾部。reverse_iterator 与 const_reverse_iterator 同样分别对应于 T * 和 const T*，所不同的是，对这两个迭代器进行递增的效果是由容器的尾部反向遍历到容器头部。



**注意**：vector::insert，对于 C++98 中，第一个参数均为 iterator；而对于 C++11 中，第一个参数均为 const_iterator。vector::erase 的情况也是这样。



## **27.** **使用 distance** **和 advance** **将容器的 const_iterator** **转换成 iterator**



```
int test_item_27()
{
	typedef std::deque<int> IntDeque;
	typedef IntDeque::iterator Iter;
	typedef IntDeque::const_iterator ConstIter;
 
	IntDeque d(5, 10);
	ConstIter ci;
	ci = d.cbegin() + 1; // 使ci指向d
	Iter i(d.begin());
	std::advance(i, std::distance<ConstIter>(i, ci));
 
	return 0;
}
```



std::distance 用以取得两个迭代器 (它们指向同一个容器) 之间的距离；std::advance 则用于将一个迭代器移动指定的距离。



std::distance 和 std::advance 的使用参考：https://blog.csdn.net/fengbingchun/article/details/77985191



## **28.** **正确理解由 reverse_iterator** **的 base()** **成员函数所产生的 iterator** **的用法**



```
int test_item_28()
{
	std::vector<int> v;
	v.reserve(5);
 
	for (int i = 1; i <= 5; ++i) v.push_back(i);
 
	std::vector<int>::reverse_iterator ri = std::find(v.rbegin(), v.rend(), 3); // 使ri指向3
	std::vector<int>::iterator i(ri.base());
	fprintf(stdout, "%d\n", (*i)); // 4
	v.insert(i, 99);
	for (auto it = v.cbegin(); it != v.cend(); ++it) fprintf(stdout, "value: %d\n", *it); // 1 2 3 99 4 5
 
	v.clear(); v.reserve(5);
	for (int i = 1; i <= 5; ++i) v.push_back(i);
	ri = std::find(v.rbegin(), v.rend(), 3);
	v.erase((++ri).base());
	for (auto it = v.cbegin(); it != v.cend(); ++it) fprintf(stdout, "value: %d\n", *it); // 1 2 4 5
 
	return 0;
}
```



如果要在一个 reverse_iterator ri 指定的位置上插入新元素，则只需在 ri.base() 位置处插入元素即可。对于插入操作而言，ri 和 ri.base() 是等价的，ri.base() 是真正与 ri 对应的 iterator。



如果要在一个 reverse_iterator ri 指定的位置上删除一个元素，则需要在 ri.base() 前面的位置上执行删除操作。对于删除操作而言，ri 和 ri.base() 是不等价的，ri.base() 不是与 ri 对应的 iterator。



## **29.** **对于逐个字符的输入请考虑使用 istreambuf_iterator**



```
int test_item_29()
{
	// 把一个文本文件的内容拷贝到一个string对象中
	std::ifstream inputFile("interestingData.txt");
	inputFile.unsetf(std::ios::skipws); // 禁止忽略inputFile中的空格
	std::string fileData((std::istream_iterator<char>(inputFile)), std::istream_iterator<char>()); // 速度慢
 
	std::string fileData2((std::istreambuf_iterator<char>(inputFile)), std::istreambuf_iterator<char>()); // 速度快
 
	return 0;
}
```



std::istream_iterator<char> 对象使用 operator>> 从输入流中读取单个字符，而 std::istreambuf_iterator<char > 则直接从流的缓冲区中读取下一个字符。std::istreambuf_iterator 不会跳过任何字符，它只是简单地取回流缓冲区中的下一个字符，而不管它们是什么字符，因此用不着清除输入流的 skipws 标志。



同样地，对于非格式化的逐个字符输出过程，你也应该考虑使用 std::ostreambuf_iterator，它可以避免因使用 std::ostream_iterator 而带来的额外负担。



## **30.** **确保目标区间足够大**



```
int transmogrify(int x) { return (x + 1); }
 
int test_item_30()
{
	std::vector<int> values{ 1, 2, 3 };
	std::vector<int> results;
	results.reserve(results.size() + values.size()); // 可避免内存的重新分配
	//std::transform(values.cbegin(), values.cend(), results.end(), transmogrify); // 错误，segmentation fault
	std::transform(values.cbegin(), values.cend(), std::back_inserter(results), transmogrify); // 正确
	// 在内部，std::back_inserter返回的迭代器将使得push_back被调用，所以back_inserter可适用于所有提供了push_back方法的容器
 
	std::list<int> results2;
	std::transform(values.cbegin(), values.cend(), std::front_inserter(results2), transmogrify);
	// std::front_inserter在内部利用了push_front，所以front_inserter仅适用于那些提供了push_front成员函数的容器
 
 
	return 0;
}
```



无论何时，如果所使用的算法需要指定一个目标区间，那么必须确保目标区间足够大，或者确保它会随着算法的运行而增大。要在算法执行过程中增大目标区间，请使用插入型迭代器，比如 ostream_iterator 或者由 back_inserter、front_inserter 和 inserter 返回的迭代器。



## **31.** **了解各种与排序有关的选择**



```
bool qualityCompare(const std::string& lhs, const std::string& rhs)
{
	return (lhs < rhs);
}
 
bool hasAcceptableQuality(const std::string& w)
{
	return true; // 判断w的质量值是否为2或者更好
}
 
int test_item_31()
{
	std::vector<std::string> vec(50, "xxx");
	std::partial_sort(vec.begin(), vec.begin() + 20, vec.end(), qualityCompare); // 将质量最好的20个元素顺序放在vec的前20个位置上
 
	std::nth_element(vec.begin(), vec.begin() + 19, vec.end(), qualityCompare); // 将最好的20个元素放在vec的前部，但并不关心它们的具体排列顺序
 
	// std::partia_sort和std::nth_element在效果上唯一不同之处在于：partial_sort对位置1--20中的元素进行了排序，而
	// nth_element没有对它们进行排序。然而，这两个算法都将质量最好的20个vec放到了矢量的前部
 
	std::vector<std::string>::iterator begin(vec.begin());
	std::vector<std::string>::iterator end(vec.end());
	std::vector<std::string>::iterator goalPosition; // 用于定位感兴趣的元素
	// 找到具有中间质量级别的string
	goalPosition = begin + vec.size() / 2; // 如果全排序的话，待查找的string应该位于中间
	std::nth_element(begin, goalPosition, end, qualityCompare); // 找到vec的中间质量值
	// 现在goalPosition所指的元素具有中间质量
 
	// 找到区间中具有75%质量的元素
	std::vector<std::string>::size_type goalOffset = 0.25 * vec.size(); // 找出如果全排序的话，待查找的string离起始处有多远
	std::nth_element(begin, begin + goalOffset, end, qualityCompare); // 找到75%处的质量值	
 
	// 将满足hasAcceptableQuality的所有元素移到前部，然后返回一个迭代器，指向第一个不满足条件的string
	std::vector<std::string>::iterator goodEnd = std::partition(vec.begin(), vec.end(), hasAcceptableQuality);
 
	return 0;
}
```



std::sort/sdk::stable_sort/std::partial_sort 使用参考：https://blog.csdn.net/fengbingchun/article/details/71305229



std::nth_element：用于排序一个区间，它使得位置 n 上的元素正好是全排序情况下的第 n 个元素。而且，当 nth_element 返回的时候，所有按全排序规则 (即 sort 的结果) 排在位置 n 之前的元素也都被排在位置 n 之前，而所有按全排序规则排在位置 n 之后的元素则都被排在位置 n 之后。



std::partial_sort 和 std::nth_element 在排列等价元素的时候，有它们自己的做法，你无法控制它们的行为。



std::partial_sort、std::nth_element 和 std::sort 都属于非稳定的排序算法，但是有一个名为 std::stable_sort 的算法可以提供稳定排序特性。



std::nth_element 除了可以用来找到排名在前的 n 个元素以外，它还可以用来找到一个区间的中间值，或者找到某个特定百分比上的值。



std::partition：可以把所有满足某个特定条件的元素放在区间的前部。



std::nth_element 和 std::partition 使用参考：https://blog.csdn.net/fengbingchun/article/details/78034969



总结排序选择：(1). 如果需要对 vector、string、deque 或者数组中的元素执行一次完全排序，那么可以使用 sort 或者 stable_sort。(2). 如果有一个 vector、string、deque 或者数组，并且只需要对等价性最前面的 n 个元素进行排序，那么可以使用 partial_sort。(3). 如果有一个 vector、string、deque 或者数组，并且需要找到第 n 个位置上的元素，或者，需要找到等价性前面的 n 个元素但又不必对这 n 个元素进行排序，那么，nth_element 正是你所需要的函数。(4). 如果需要将一个标准序列容器中的元素按照是否满足某个特定的条件区分开来，那么，partition 和 stable_partition 可能正是你所需要的。(5). 如果你的数据在一个 list 种，那么你仍然可以直接调用 partition 和 stable_partition 算法；你可以用 list::sort 来替代 sort 和 stable_sort 算法。但是，如果你需要获得 partial_sort 或 nth_element 算法的效果，那么，你可以有一些间接的途径来完成这项任务。



## **32.** **如果确实需要删除元素，则需要在 remove** **这一类算法之后调用 erase**



```
int test_item_32()
{
	std::vector<int> v;
	v.reserve(10);
	for (int i = 1; i <= 10; ++i) v.push_back(i);
	fprintf(stdout, "v.size: %d\n", v.size()); // 输出10
	v[3] = v[5] = v[9] = 99;
	std::remove(v.begin(), v.end(), 99); // 删除所有值等于99的元素
	fprintf(stdout, "v.size: %d\n", v.size()); // 仍然输出10, remove不是真正意义上的删除，因为它做不到
	for (auto i : v) fprintf(stdout, "%d\n", i);
 
	v.erase(std::remove(v.begin(), v.end(), 99), v.end()); // 真正删除所有值等于99的元素	
 
	return 0;
}
```



std::remove 使用参考：https://blog.csdn.net/fengbingchun/article/details/78034969



std::remove 并不接受容器作为参数，所以 remove 并不知道这些元素被存放在哪个容器中。并且，remove 也不可能推断出是什么容器，因为无法从迭代器推知对应的容器类型。因为从容器中删除元素的唯一方法是调用该容器的成员函数，而 remove 并不知道它操作的元素所在的容器，所以 remove 不可能从容器中删除元素。



std::list 的 remove 成员函数是 STL 中唯一一个名为 remove 并且确实删除了容器中元素的函数。



std::remove 并不是唯一一个适用于这种情形的算法，其它还有两个属于”remove 类”的算法：remove_if 和 unique。如同 list::remove 会真正删除元素 (并且比使用 erase-remove 习惯用法更为高效) 一样，std::list::unique 也会真正删除元素(而且比使用 erase-unique 更为高效)。



## **33.** **对包含指针的容器使用 remove** **这一类算法时要特别小心**



```
class Widget33 {
public:
	bool isCertified() const { return true; }
};
 
// 如果*pWidget是一个未被验证的Widget33,则删除该指针，并把它置成空
void delAndNullifyUncertified(Widget33*& pWidget)
{
	if (!pWidget->isCertified()) {
		delete pWidget;
		pWidget;
	}
}
 
int test_item_33()
{
	std::vector<Widget33*> v;
	for (int i = 0; i < 5; ++i) v.push_back(new Widget33);
 
	// 删除那些指向未被验证过的Widget33对象的指针，会资源泄露
	v.erase(std::remove_if(v.begin(), v.end(), std::not1(std::mem_fun(&Widget33::isCertified))), v.end());
 
	// 一种可以消除资源泄露的做法
	// 将所有指向未被验证的Widget33对象的指针删除并置成空
	std::for_each(v.begin(), v.end(), delAndNullifyUncertified);
	// 删除v中的空指针，必须将0转换成一个指针，这样C++才能正确推断出remove的第三个参数类型
	v.erase(std::remove(v.begin(), v.end(), static_cast<Widget33*>(0)), v.end());
 
	// 使用智能指针可防止资源泄露
	std::vector<std::shared_ptr<Widget33>> v2;
	for (int i = 0; i < 5; ++i) v2.push_back(std::make_shared<Widget33>());
	// 下面语句需要编译器必须能够把智能指针类型std::shared<Widget33>隐式转换为对应的内置指针类型Widget33*才能通过编译
	//v2.erase(std::remove_if(v2.begin(), v2.end(), std::not1(std::mem_fun(&Widget33::isCertified))), v2.end());
 
	return 0;
}
```



当容器中存放的是指向动态分配的对象的指针的时候，应该避免使用 remove 和类似的算法 (remove_if 和 unique)。



如果容器中存放的不是普通指针，而是具有引用计数功能的智能指针，那么就可以直接使用 erase-remove 的习惯用法。



## **34.** **了解哪些算法要求使用排序的区间作为参数**



并非所有的算法都可以应用于任何区间。举例来说，remove 算法要求单向迭代器并且要求可以通过这些迭代器向容器中的对象赋值。所以，它不能用于由输入迭代器指定的区间，也不适用于 map 或 multimap，同样不适用于某些 set 和 multiset 的实现。同样地，很多排序算法要求随机访问迭代器，所以对于 list 的元素不可能调用这些算法。有些算法要求排序的区间，即区间中的值是排过序的。有些算法既可以与排序的区间一起工作，也可以与未排序的区间一起工作，但是当它们作用在排序的区间上时，算法会更加有效。



要求排序区间的 STL 算法：binaray_search、lower_bound、upper_bound、equal_range、set_union、set_intersection、set_difference、set_symmetric_difference、merge、inplace_merge、includes。



unique、unique_copy 并不一定要求排序的区间，但通常情况下会与排序区间一起使用。



用于查找的算法 binaray_search、lower_bound、upper_bound、equal_range 要求排序的区间，因为它们用二分法查找数据。实际上，这些算法并不一定保证对数时间的查找效率。只有当它们接受了随机访问迭代器的时候，它们才保证有这样的效率。



set_union、set_intersection、set_difference、set_symmetric_difference 这 4 个算法提供了线性时间效率的集合操作。如果它们不满足排序的区域的要求，它们就无法在线性时间内完成工作。



merge 和 inplace_merge 实际上实现了合并和排序的联合操作：它们读入两个排序的区间，然后合并成一个新的排序区间，其中包含了原来两个区间中的所有元素。它们具有线性时间的性能，但如果它们不知道源区间已经排过序的话，它们就不可能在线性时间内完成。



最后一个要求排序源区间的算法是 includes，它可用来判断一个区间中的所有对象是否都在另一个区间中。因为 includes 总是假设这两个区间是排序的，所以它承诺线性时间的效率。如果没有这一前提的话，它通常会运行得更慢。



unique 和 unique_copy 与上述讨论过的算法有所不同，它们即使对于未排序的区间也有很好的行为。unique 使用了与 remove 类似的办法来删除区间中的元素，而并非真正意义上的删除。



**如果你为一个算法提供了一个排序的区间，而这个算法也带一个比较函数作为参数，那么，你一定要保证你传递的比较函数与这个排序区间所用的比较函数有一致的行为**。



以上这些算法在 include<algorithm> 中，使用参考：https://blog.csdn.net/fengbingchun/article/details/78034969



## **35.** **通过 mismatch** **或 lexicographical_compare** **实现简单的忽略大小写的字符串比较**



```
// 忽略大小写地比较字符c1和c2，如果c1<c2,返回-1；如果c1>c2,返回1；如果c1==c2,返回0
int ciCharCompare(char c1, char c2)
{
	int lc1 = std::tolower(static_cast<unsigned char>(c1));
	int lc2 = std::tolower(static_cast<unsigned char>(c2));
 
	if (lc1 < lc2) return -1;
	if (lc1 > lc2) return 1;
	return 0;
}
 
int ciStringCompareImpl(const std::string& s1, const std::string& s2)
{
	typedef std::pair<std::string::const_iterator, std::string::const_iterator> PSCI;
	PSCI p = std::mismatch(s1.begin(), s1.end(), s2.begin(), std::not2(std::ptr_fun(ciCharCompare)));
 
	if (p.first == s1.end()) { // 如果为true,要么s1和s2相等，或者s1比s2短
		if (p.second == s2.end()) return 0;
		else return -1;
	}
 
	return ciCharCompare(*p.first, *p.second); // 字符串之间的关系和这两个不匹配的字符之间的关系相同
}
 
int ciStringCompare(const std::string& s1, const std::string& s2)
{
	// 把短的字符串作为第一个区间传入
	if (s1.size() <= s2.size()) return ciStringCompareImpl(s1, s2);
	else return -ciStringCompareImpl(s2, s1);
}
 
// 返回在忽略大小写的情况下，c1是否在c2之前
bool ciCharLess(char c1, char c2)
{
	return std::tolower(static_cast<unsigned char>(c1)) <
		std::tolower(static_cast<unsigned char>(c2));
}
 
bool ciStringCompare2(const std::string& s1, const std::string& s2)
{
	return std::lexicographical_compare(s1.begin(), s1.end(), s2.begin(), s2.end(), ciCharLess);
}
 
bool ciStringCompare3(const std::string& s1, const std::string& s2)
{
	// 前提：不考虑国际化支持，也确定字符串中不会包含内嵌的空字符
	return strcmp(s1.c_str(), s2.c_str());
}
 
int test_item_35()
{
	std::string str1{ "xxz" }, str2{ "xxx" };
	fprintf(stdout, "str1:str2: %d\n", ciStringCompare(str1, str2));
	fprintf(stdout, "str1:str2: %d\n", ciStringCompare2(str1, str2));
	fprintf(stdout, "str1:str2: %d\n", ciStringCompare3(str1, str2));
 
	return 0;
}
```



std::lexicographical_compare 是 strcmp 的一个泛化版本。不过，strcmp 只能与字符数组一起工作，而 lexicographical_compare 则可以与任何类型的值的区间一起工作。而且，strcmp 总是通过比较两个字符来判断它们的关系相等、小于还是大于，而 lexicographical_compare 则可以接受一个判别式，由该判别式来决定两个值是否满足一个用户自定义的准则。



strcmp 通常是被优化过的，它们在字符串的处理上一般要比通用算法 mismatch 和 lexicographical_compare 快。



## **36.** **理解 copy_if** **算法的正确实现**



```
int test_item_36()
{
	std::vector<int> v1{ 1, 2, 3, 4, 5 }, v2(v1.size());
 
	auto it = std::copy_if(v1.begin(), v1.end(), v2.begin(), [](int i) { return (i % 2 == 1); });
	v2.resize(std::distance(v2.begin(), it));
 
	for (const auto& v : v2)
		fprintf(stdout, "%d\n", v); // 1 3 5
 
	return 0;
}
```



C++11 中增加了 std::copy_if 函数。



## **37.** **使用 accumulate** **或者 for_each** **进行区间统计**



```
// 接受当前的长度总和值和新的字符串，然后返回更新之后的总和值
std::string::size_type stringLengthSum(std::string::size_type sumSoFar, const std::string& s)
{
	return sumSoFar + s.size();
}
 
struct Point {
	Point(double initX, double initY) : x(initX), y(initY) {}
	double x, y;
};
 
class PointAverage : public std::unary_function<Point, void> {
public:
	PointAverage() : xSum(0), ySum(0), numPoints(0) {}
 
	void operator()(const Point& p)
	{
		++numPoints;
		xSum += p.x;
		ySum += p.y;
	}
 
	Point result() const
	{
		return Point(xSum / numPoints, ySum / numPoints);
	}
 
private:
	size_t numPoints;
	double xSum, ySum;
};
 
int test_item_37()
{
	std::vector<double> v{ 1.0f, 2.2f, 3.3f, 4.5f };
	double sum = std::accumulate(v.cbegin(), v.cend(), 0.0); // 注意：初始值被指定为0.0,而不是简单的0
	fprintf(stdout, "vaule: %f\n", sum); // 11.000000
 
	fprintf(stdout, "sum of the ints: %d\n", std::accumulate(std::istream_iterator<int>(std::cin), std::istream_iterator<int>(), 0)); // 输入非整数值结束,如字母
 
	std::set<std::string> ss{ "abc", "cde", "xyzw" };
	// 计算一个容器中字符串的长度总和
	std::string::size_type lengthSum = std::accumulate(ss.cbegin(), ss.cend(),
		static_cast<std::string::size_type>(0), stringLengthSum);
	fprintf(stdout, "length sum: %d\n", lengthSum); // 10
 
	// 计算一个区间中数值的乘积
	std::vector<float> vf{ 1.f, 2.f, 3.f, 1.5f };
	float product = std::accumulate(vf.cbegin(), vf.cend(), 1.f, std::multiplies<float>());
	fprintf(stdout, "product: %f\n", product); // 9.000000
 
	// 计算出一个区间中所有点的平均值
	std::list<Point> lp{ { 1, 2 }, { 2, 3 }, { 3, 4 }, { 4, 5 } };
	Point avg = std::for_each(lp.cbegin(), lp.cend(), PointAverage()).result();
 
	return 0;
}
```



std::accumulate 有两种形式：第一种形式有两个迭代器和一个初始值，它返回该初始值加上由迭代器标识的区间中的值的总和。



std::accumulate 只要求输入迭代器，所以你可以使用 std::istream_iterator 和 std::istreambuf_iterator。



std::accumulate 的第二种形式带一个初始值和一个任意的统计函数。



std::for_each 是另一个可被用来统计区间的算法，而且它不受 accumulate 的那些限制。如同 accumulate 一样，for_each 也带两个参数：一个是区间，另一个是函数 (通常是函数对象)---- 对区间中的每个元素都要调用这个函数，但是，传给 for_each 的这个函数只接收一个实参(即当前的区间元素)。for_each 执行完毕后会返回它的函数。(实际上，它返回的是这个函数的一份拷贝。) 重要的是，传给 for_each 的函数 (以及后来返回的函数) 可以有副作用。



std::for_each 和 std::accumulate 在两个方面有所不同：首先，名字 accumulate 暗示着这个算法将会计算出一个区间的统计信息。而 for_each 听起来就好像是对一个区间的每个元素做一个操作。用 for_each 来统计一个区间是合法的，但是不如 accumulate 来得清晰。其次，accumulate 直接返回我们所要的统计结果，而 for_each 却返回一个函数对象，我们必须从这个函数对象中提取出我们所要的统计信息。在 C++ 中，这意味着我们必须在函数子类中 加入一个成员函数，以便获得我们想要的统计信息。



## **38.** **遵循按值传递的原则来设计函数子类**



```
class Widget38 {};
template<typename T> class BPFC;
 
template<typename T>
class BPFCImpl : public std::unary_function<T, void> {
private:
	Widget38 w; // 原来BPFC中所有数据现在都放在这里
	int x;
	virtual ~BPFCImpl(); // 多态类需要虚析构函数
	virtual void operator()(const T& val) const;
 
	friend class BPFC<T>; // 允许BPFC访问内部数据
};
 
template<typename T>
class BPFC : public std::unary_function<T, void> { // 新的BPFC类:短小、单态
private:
	BPFCImpl<T>* pImpl; // BPFC唯一的数据成员
public:
	void operator()(const T& val) const // 现在这是一个非虚函数，将调用转到BPFCImpl中
	{
		pImpl->operator()(val);
	}
};
```



无论是 C 还是 C++，都不允许将一个函数作为参数传递给另一个函数，相反，你必须传递函数指针。C 和 C++ 的标准库函数都遵循这一规则：**函数指针是按值传递的**。



STL 函数对象是函数指针的一种抽象和建模形式，所以，按照惯例，在 STL 中，函数对象在函数之间来回传递的时候也是按值传递 (即被拷贝) 的。标准库中一个最好的证明是 for_each 算法，它需要一个函数对象作为参数，同时其返回值也是一个函数对象，而且都是按值传递的。



由于函数对象往往会按值传递和返回，所以，你必须确保你编写的函数对象在经过了传递之后还能正常工作。这意味着两件事：首先，你的函数对象必须尽可能地小，否则拷贝的开销会非常昂贵；其次，函数对象必须是单态的 (不是多态的)，也就是说，它们不得使用虚函数。这是因为，如果参数的类型是基类类型，而实参是派生类对象，那么在传递过程中会产生剥离问题 (slicing problem)：在对象拷贝过程中，派生部分可能会被去掉，而仅保留了基类部分。



既允许函数对象可以很大并且 / 或者保留多态性，又可以与 STL 所采用的按值传递函数子的习惯保持一致的一种两全其美的办法：将所需的数据和虚函数从函数子类中分离出来，放到一个新的类中；然后在函数子类中包含一个指针，指向这个新类的对象。



## **39.** **确保判别式是”** **纯函数”**



一个判别式 (predicate) 是一个返回值为 bool 类型 (或者可以隐式地转换为 bool 类型) 的函数。在 STL 中，判别式有着广泛的用途。标准关联容器的比较函数就是判别式；对于像 find_if 以及各种与排序有关的算法，判别式往往也被作为参数来传递。



一个纯函数 (pure function) 是指返回值仅仅依赖于其参数的函数。在 C++ 中，纯函数所能访问的数据应该仅局限参数以及常量(在函数生命期内不会被改变，自然地，这样的常量数据应该被声明为 const)。如果一个纯函数需要访问那些可能在两次调用之间发生变化的数据，那么用相同的参数在不同的时刻调用该函数就有可能会得到不同的结果，这将与纯函数的定义相矛盾。



判别式类 (predicate class) 是一个函数的子类，它的 operator()函数是一个判别式，也就是说，它的 operator()返回 true 或者 false。STL 中凡是能接受判别式的地方，就既可以接受一个真正的判别式，也可以接受一个判别式类的对象。



## **40.** **若一个类是函数子，则应使它可配接**



```
bool isInteresting(const std::string* pw) { return true; }
 
int test_item_40()
{
	std::list<std::string*> strs{ new std::string("xxx"), new std::string("yyy"), new std::string("zzz") };
	std::list<std::string*>::iterator it = std::find_if(strs.begin(), strs.end(), isInteresting);
	std::list<std::string*>::iterator it2 = std::find_if(strs.begin(), strs.end(), std::not1(std::ptr_fun(isInteresting)));
 
	return 0;
}
```



std::ptr_fun 仅完成一些类型定义的工作，这些类型定义是 std::not1 所必须的。在 C++11 中已不推荐使用 std::ptr_fun。



4 个标准的函数配接器 (not1、not2、bind1st、bind2nd，后两个，在 C++11 中已不推荐使用) 都要求一些特殊的类型定义，那些非标准的、与 STL 兼容的配接器通常也是如此 (例如，Boost 提供的 STL 中就包含了这样的组件)。提供了这些必要的类型定义的函数对象被称为可配接的(adaptable) 函数对象，反之，如果函数对象缺少这些类型定义，则称为不可配接的。可配接的函数对象能够与其它 STL 组件更为默契地协同工作，它们能够应用于更多的上下文环境中，因此你应当尽可能地使你编写的函数对象可以配接。



“这些特殊的类型定义” 是 argument_type、first_argument_type、second_argument_type 以及 result_type。不同种类的函数子类所需提供的类型定义也不尽相同，它们是这些名字的不同子集。提供这些类型定义最简便的办法是让函数子从特定的基类继承，或者更准确地说，从一个基结构继承。如果函数子类的 operator() 只有一个实参，那么它应该从 std::unary_function 继承；如果函数子类的 operator() 有两个实参，那么它应该从 std::binary_function 继承。C++11 中已不推荐使用这两个函数。



STL 函数对象是 C++ 函数的一种抽象和建模形式，而每个 C++ 函数只有一组确定的参数类型和一个返回类型。所以，STL 总是假设每个函数子类只有一个 operator() 成员函数，并且其参数和返回类型应该吻合 unary_function 或 binary_function 的模板参数。



## **41.** **理解 ptr_fun****、men_fun** **和 mem_fun_ref** **的来由**



```
class Widget41 {
public:
	void test() {}
};
 
void test(Widget41& w) {}
 
int test_item_41()
{
	std::vector<Widget41> vw;
 
	std::for_each(vw.begin(), vw.end(), test);
	std::for_each(vw.begin(), vw.end(), std::ptr_fun(test));
	std::for_each(vw.begin(), vw.end(), std::mem_fun_ref(&Widget41::test));
 
	std::list<Widget41*> lpw;
	std::for_each(lpw.begin(), lpw.end(), std::mem_fun(&Widget41::test));
 
	return 0;
}
```



std::ptr_fun：将函数指针转换为函数对象。



std::mem_fun：将成员函数转换为函数对象 (指针版本)。



std::mem_fun_ref：将成员函数转换为函数对象 (引用版本)。



像 mem_fun_t 这样的类被称为函数对象配接器 (function object adapter)。这些由 mem_fun 和 mem_fun_ref 产生的对象不仅使得 STL 组件可以通过同一种语法形式来调用所有的函数，而且它们还提供了一些重要的类型定义，就像 fun_ptr 所产生的对象一样。



如果你对什么时候该使用 ptr_fun，什么时候不该使用 ptr_fun 感到困惑，那么你可以在每次将一个函数传递给一个 STL 组件的时候总是使用它。STL 不会在意的，而且这样做也不会带来运行时的性能损失。另一种策略是，只有在迫不得已的时候才使用 ptr_fun。如果你省略了那些必要的类型定义，编译器就会提醒你，然后你再回去把 ptr_fun 加上。



mem_fun 和 mem_fun_ref 的情形则截然不同。每次在将一个成员函数传递给一个 STL 组件的时候，你就要使用它们。因为它们不仅仅引入了一些类型定义 (可能是必要的，也可能不是必要的)，而且它们还转换调用语法的形式来适应算法 ---- 它们将针对成员函数的调用语法转换为 STL 组件所使用的调用语法。



C++11 中已不再推荐使用 ptr_fun、mem_fun、mem_fun_ref 等相关函数。



## **42.** **确保 less<T>** **与 operator<** **具有相同的语义**



应该尽量避免修改 less 的行为，因为这样做很可能会误导其他的程序员。如果你使用了 less，无论是显式地或是隐式地，你都需要确保它与 operator < 具有相同的意义。如果你希望以一种特殊的方式来排序对象，那么最好创建一个特殊的函数子类，它的名字不能是 less。



## **43.** **算法调用优先于手写的循环**



理由：(1). 效率：算法通常比程序员自己写的循环效率更高。(2). 正确性：自己写循环比使用算法更容易出错。(3). 可维护性：使用算法的代码通常比手写循环的代码更加简洁明了。



如果你要做的工作与一个算法所实现的功能很相近，那么用算法调用更好。但是如果你的循环很简单，而若使用算法来实现的话，却要求混合使用绑定器和配接器或者要求一个单独的函数子类，那么，可能使用手写的循环更好。最后，如果你在循环中要做的工作很多，而且又很复杂，则最好使用算法调用。



## **44.** **容器的成员函数优先于同名的算法**



有些 STL 容器提供了一些与算法同名的成员函数。比如，关联容器提供了 count、find、lower_bound、upper_bound 和 equal_range，而 list 则提供了 remove、remove_if、unique、sort、merge 和 reverse。大多数情况下，你应该使用这些成员函数，而不是相应的 STL 算法。这里有两个理由：第一，成员函数往往速度快；第二，成员函数通常与容器 (特别是关联容器) 结合得更加紧密，这是算法所不能比的。原因在于，算法和成员函数虽然有同样的名称，但是它们所做的事情往往不完全相同。



对于标准的关联容器，选择成员函数而不选择对应的同名算法，这可以带来几方面的好处：第一，你可以获得对数时间的性能，而不是线性时间的性能。第二，你可以使用等价性来确定两个值是否”相同”，而等价性是关联容器的一个本质定义。第三，你在使用 map 和 multimap 的时候，将很自然地只考虑元素的键部分，而不是完整的 (key, value) 对。



remove、remove_if、unique、sort、merge 以及 reverse 这些算法无一例外地需要拷贝 list 容器中的对象，而专门为 list 容器量身定做的成员函数则无需任何对象拷贝，它们只是简单地维护好那些将 list 节点连接起来的指针。list 成员函数的行为不同于与其同名的算法的行为。如果真的要从一个容器中删除对象的话，你在调用了 remove、remove_if 或者 unique 算法之后，必须紧接着再调用 erase；而 list 的 remove、remove_if 和 unique 成员函数则实实在在地删除了元素，所以你无需再调用 erase 了。在 sort 算法与 list 的 sort 函数之间有一个很重要的区别是，前者根本不能被应用到 list 容器上，这是因为，list 的迭代器是双向迭代器，而 sort 算法要求随机访问迭代器。在 merge 算法和 list 的 merge 函数之间也存在行为上的隔阂：merge 算法是不允许修改其源区间的，而 list::merge 则总是在修改它所操作的链表。



## **45.** **正确区分 count****、find****、binary_search****、lower_bound****、upper_bound** **和 equal_range**



在选择具体的查找策略时, 由迭代器指定的区间是否是排序的，这是一个至关重要的决定条件。如果区间是排序的，那么通过 binary_search、lower_bound、upper_bound 和 equal_range，你可以获得更快的查找速度 (通常是对数时间的效率)。如果迭代器并没有指定一个排序的区间，那么你的选择范围将局限于 count、count_if、find 以及 find_if，而这些算法仅能提供线性时间的效率。





![img](图表库/20191124142100772.png)





以上这些函数用法参考：https://blog.csdn.net/fengbingchun/article/details/78034969



## **46.** **考虑使用函数对象而不是函数作为 STL** **算法的参数**



```
struct StringSize : public std::unary_function<std::string, std::string::size_type> {
	std::string::size_type operator()(const std::string& s) const
	{
		return s.size();
	}
};
 
int test_item_46()
{
	std::set<std::string> s{ "abc", "cde", "xyzw" };
	std::transform(s.begin(), s.end(), std::ostream_iterator<std::string::size_type>(std::cout, "\n"), std::mem_fun_ref(&std::string::size)); // 3 3 4，普通函数
 
	std::transform(s.begin(), s.end(), std::ostream_iterator<std::string::size_type>(std::cout, "\n"), StringSize()); // 3 3 4, 函数对象
 
	return 0;
}
```



在 C/C++ 中并不能真正地将一个函数作为参数传递给另一个函数。如果我们试图将一个函数作为参数进行传递，则编译器会隐式地将它转换成一个指向该函数的指针，并将该指针传递过去。函数指针参数抑制了内联机制。



以函数对象作为 STL 算法的参数，这种做法提供了包括效率在内的多种优势。从代码被编译器接受的程度而言，它们更加稳定可靠。当然，普通函数在 C++ 中也是非常实用的，但是就有效使用 STL 而言，函数对象通常更加实用一些。



## **47.** **避免产生”** **直写型”(write-only)** **的代码**



当你编写代码的时候，它看似非常直接和简捷，因为它是由某些基本想法 (比如，erase-remove 习惯用法加上在 find 中使用 reverse_interator 的概念) 自然而形成的。然而，阅读代码的人却很难将最终的语句还原成它所依据的思路，这就是”直写型的代码”叫法的来历：虽然很容易编写，但是难以阅读和理解。一段代码是否是”直写型”取决于其读者的知识水平。



## **48.** **总是包含 (#include)** **正确的头文件**



C++ 标准与 C 的标准有所不同，它没有规定标准库中的头文件之间的相互包含关系。



总结每个与 STL 有关的标准头文件中所包含的内容：



(1). 几乎所有的标准 STL 容器都被声明在与之同名的头文件中，比如 vector 被声明在 < vector > 中，list 被声明在 < list > 中，等等。但是 < set > 和 < map > 是个例外，<set > 中声明了 set 和 multiset，<map > 中声明了 map 和 multimap。



(2). 除了 4 个 STL 算法以外，其它所有的算法都被声明在 < algorithm > 中，这 4 个算法使 accumulate、inner_product、adjacent_difference 和 partial_sum，它们被声明在头文件 < numeric > 中。



(3). 特殊类型的迭代器，包括 istream_iterator 和 istreambuf_iterator，被声明在 < iterator > 中。



(4). 标准的函数子 (比如 less<T>) 和函数子配接器 (比如 not1、bind2nd) 被声明在头文件 < functional > 中。



任何时候如果你使用了某个头文件中的一个 STL 组件，那么你就一定要提供对应的 #include 指令，即使你正在使用的 STL 平台允许你省略 #include 指令，你也要将它们包含到你的代码中。当你需要将代码移植到其它平台上的时候，移植的压力就会减轻。



## **49.** **学会分析与 STL** **相关的编译器诊断信息**



在一个被声明为 const 的成员函数内部，该类的所有非静态数据成员都自动被转变为相应的 const 类型。



一些技巧：



(1). vector 和 string 的迭代器通常就是指针，所以当错误地使用了 iterator 的时候，编译器的诊断信息中可能会引用到指针类型。例如，如果源代码中引用了 vector<double>::iterator，那么编译器的诊断信息中极有可能就会提及 double * 指针。



(2). 如果诊断信息中提到了 back_insert_iterator、front_insert_iterator 或者 insert_iterator，则几乎总是意味着你错误地调用了 back_inserter、front_inserter 或者 inserter。如果你并没有直接调用这些函数，则一定是你所调用的某个函数直接或者间接地调用了这些函数。



(3). 类似地，如果诊断信息中提到了 binder1st 或者 binder2nd，那么你可能是错误地使用了 bind1st 和 bind2nd。



(4). 输出迭代器 (如 ostream_iterator、ostreambuf_iterator 以及那些由 back_inserter、front_inserter、front_inserter 和 inserter 函数返回的迭代器) 在赋值操作符内部完成其输出或者插入工作，所以，如果在使用这些迭代器的时候犯了错误，那么你所看到的错误消息中可能会提到与赋值操作符有关的内容。



(5). 如果你得到的错误消息来源于某一个 STL 算法的内部实现 (例如，引起错误的源代码在 < algorithm > 中)，那也许是你在调用算法的时候使用了错误的类型。例如，你可能使用了不恰当的迭代器类型。



(6). 如果你正在使用一个很常见的 STL 组件，比如 vector、string 或者 for_each 算法，但是从错误消息来看，编译器好像对此一无所知，那么可能是你没有包含相应的头文件。



## **50.** **熟悉与 STL** **相关的 Web** **站点**



书中介绍了 SGI STL(已废弃)、STLport、Boost。



个人强烈推荐：http://www.cplusplus.com/ 尤其是其中的 Reference(http://www.cplusplus.com/reference/)，有时 cplusplus 网站会打不开，此时也可参考 cppreference：https://en.cppreference.com/w/cpp



**GitHub**：https://github.com/fengbingchun/Messy_Test

