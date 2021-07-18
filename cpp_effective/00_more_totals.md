# More Effective C++

[More Effective C++35个改善编程与设计的有效方法笔记_网络资源是无限的-CSDN博客](https://blog.csdn.net/fengbingchun/article/details/102990753)

## 1. 指针与引用的区别

```
void printDouble(const double& rd)
{
	std::cout<<rd; // 不需要测试rd,它肯定指向一个double值
}
 
void printDouble(const double* pd)
{
	if (pd) { // 检查是否为NULL
		std::cout<<*pd;
	}
}
 
int test_item_1()
{
	char* pc = 0; // 设置指针为空值
	char& rc = *pc; // 让指针指向空值，这是非常有害的，结果将是不确定的
 
	//std::string& rs; // 错误，引用必须被初始化
	std::string s("xyzzy");
	std::string& rs = s; // 正确,rs指向s
	std::string* ps; // 未初始化的指针，合法但危险
 
{
	std::string s1("Nancy");
	std::string s2("Clancy");
	std::string& rs = s1; // rs引用s1
	std::string* ps = &s1; // ps指向s1
	rs = s2; // rs仍旧引用s1,但是s1的值现在是"Clancy"
	ps = &s2; // ps现在指向s2,s1没有改变
}
 
	std::vector<int> v(10);
	v[5] = 10; // 这个被赋值的目标对象就是操作符[]返回的值，如果操作符[]
		   // 返回一个指针，那么后一个语句就得这样写: *v[5] = 10;
 
	return 0;
}
```

指针与引用看上去完全不同(指针用操作符”*”和”->”，引用使用操作符”.”)，但是它们似乎有相同的功能。指针和引用都是让你间接引用其它对象。

在任何情况下都不能使用指向空值的引用。一个引用必须总是指向某些对象。在C++里，引用应被初始化。

不存在指向空值的引用这个事实意味着使用引用的代码效率比使用指针的要高。因为在使用引用之前不需要测试它的合法性。

指针与引用的另一个重要的不同是指针可以被重新赋值以指向另一个不同的对象。但是引用则总是指向在初始化时被指定的对象，以后不能改变。

总的来说，在以下情况下你应该使用指针，一是你考虑到存在不指向任何对象的可能(在这种情况下，你能够设置指针为空)，二是你需要能够在不同的时刻指向不同的对象(在这种情况下，你能改变指针的指向)。如果总是指向一个对象并且一旦指向一个对象后就不会改变指向，那么你应该使用引用。

当你知道你必须指向一个对象并且不想改变其指向时，或者在重载操作符并为防止不必要的语义误解时(最普通的例子是操作符[])，你不应该使用指针。而在除此之外的其它情况下，则应使用指针。

关于引用的更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/69820184

## 2. 尽量使用C++风格的类型转换

```
class Widget {
public:
	virtual void func() {}
};
 
class SpecialWidget : public Widget {
public:
	virtual void func() {}
};
 
void update(SpecialWidget* psw) {}
void updateViaRef(SpecialWidget& rsw) {}
 
typedef void (*FuncPtr)(); // FuncPtr是一个指向函数的指针
int doSomething() { return 1; };
 
int test_item_2()
{
	int firstNumber = 1, secondNumber = 1;
	double result1 = ((double)firstNumber) / secondNumber; // C风格
	double result2 = static_cast<double>(firstNumber) / secondNumber; // C++风格类型转换
 
	SpecialWidget sw; // sw是一个非const对象
	const SpecialWidget& csw = sw; // csw是sw的一个引用，它是一个const对象
	//update(&csw); // 错误，不能传递一个const SpecialWidget*变量给一个处理SpecialWidget*类型变量的函数
	update(const_cast<SpecialWidget*>(&csw)); // 正确，csw的const显示地转换掉(csw和sw两个变量值在update函数中能被更新)
	update((SpecialWidget*)&csw); // 同上，但用了一个更难识别的C风格的类型转换
 
	Widget* pw = new SpecialWidget;
	//update(pw); // 错误，pw的类型是Widget*，但是update函数处理的是SpecialWidget*类型
	//update(const_cast<SpecialWidget*>(pw)); // 错误，const_cast仅能被用在影响constness or volatileness的地方，不能用在向继承子类进行类型转换
 
	Widget* pw2 = nullptr;
	update(dynamic_cast<SpecialWidget*>(pw2)); // 正确，传递给update函数一个指针是指向变量类型为SpecialWidget的pw2的指针， 如果pw2确实指向一个对象，否则传递过去的将是空指针
 
	Widget* pw3 = new SpecialWidget;
	updateViaRef(dynamic_cast<SpecialWidget&>(*pw3)); // 正确，传递给updateViaRef函数SpecailWidget pw3指针，如果pw3确实指向了某个对象，否则将抛出异常
 
	//double result3 = dynamic_cast<double>(firstNumber) / secondNumber; // 错误，没有继承关系
	const SpecialWidget sw4;
	//update(dynamic_cast<SpecialWidget*>(&sw4)); // 错误，dynamic_cast不能转换掉const
 
	FuncPtr funcPtrArray[10]; // funcPtrArray是一个能容纳10个FuncPtr指针的数组
	//funcPtrArray[0] = &doSomething; // 错误，类型不匹配
	funcPtrArray[0] = reinterpret_cast<FuncPtr>(&doSomething); // 转换函数指针的代码是不可移植的(C++不保证所有的函数指针都被用一样的方法表示)，在一些情况下这样的转换会产生不正确的结果，所以应该避免转换函数指针类型
 
	return 0;
}
```

C++通过引进四个新的类型转换(cast)操作符克服了C风格类型转换的缺点(过于粗鲁，能允许你在任何类型之间进行转换；C风格的类型转换在程序语句中难以识别)，这四个操作符是：static_cast、const_cast、dynamic_cast、reinterpret_cast。

static_cast在功能上基本上与C风格的类型转换一样强大，含义也一样。它也有功能上限制。例如，不能用static_cast像用C 风格的类型转换一样把struct转换成int类型或者把double类型转换成指针类型，另外，static_cast不能从表达式中去除const属性，因为另一个新的类型转换操作符const_cast有这样的功能。

const_cast用于类型转换掉表达式的const或volatileness属性。如果你试图使用const_cast来完成修改constness或者volatileness属性之外的事情，你的类型转换将被拒绝。

dynamic_cast被用于安全地沿着类的继承关系向下进行类型转换。这就是说，你能用dynamic_cast把指向基类的指针或引用转换成指向其派生类或其兄弟类的指针或引用，而且你能知道转换是否成功。失败的转换将返回空指针(当对指针进行类型转换时)或者抛出异常(当对引用进行类型转换时)。dynamic_cast在帮助你浏览继承层次上是有限制的，它不能被用来缺乏虚函数的类型上，也不能用它来转换掉constness。如你想在没有继承关系的类型中进行转换，你可能想到static_cast。如果是为了去除const，你总得用const_cast。

reinterpret_cast使用这个操作符的类型转换，其转换结果几乎都是执行期定义(implementation-defined)。因此，使用reinterpret_cast的代码很难移植。此操作符最普通的用途就是在函数指针之间进行转换。

关于类型转换更多介绍参考：https://blog.csdn.net/fengbingchun/article/details/51235498

## 3. 不要对数组使用多态

### 取内容问题

考虑：

```
class BST {};
class BalancedBST : public BST {};

void printBSTArray(ostream& o, const BST arr[], int num) {
	for (int i = 0; i < num; ++i) {
		out << arr[i];               // BST 定义了 operator<<操作符
	}
}
```

如下使用：

```
BST bstsArr[10];
...
printBSTArray(cout, bstsArr, 10);
```

传递给函数BST数组，自然是没问题的。

在考虑如下：

```
BalancedBST balanceBsts[10];
...
printBSTArray(cout, balanceBsts, 10);       // 能否正常运行
```

编译器确实不会报错，然是通过循环取数组内容，却有问题。arr[i]的操作相当于`*(arr + i)`，arr是个指针，指向数组起始位置，取元素时需要移动多远的距离？通过`i*sizeof(数组的对象)`来获得。

然而这里传递给函数参数的数组声明的是base class对象，所以也会以这个来计算大小。而如果传递的是derived对象的数组，则再按照base class计算大小就是错误的。

### delete删除问题

```
class BST {
public:
	virtual ~BST() { fprintf(stdout, "BST::~BST\n"); }
private:
	int score;
};
 
class BalancedBST : public BST {
public:
	virtual ~BalancedBST() { fprintf(stdout, "BalancedBST::~BalancedBST\n"); }
private:
	int length;
	int size; // 如果增加此一个int成员，执行test_item_3会segmentation fault，注释掉此变量，运行正常
};
 
int test_item_3()
{
	fprintf(stdout, "BST size: %d\n", sizeof(BST)); // 16
	fprintf(stdout, "BalancedBST size: %d\n", sizeof(BalancedBST)); // 24
 
	BST* p = new BalancedBST[10];
	delete [] p; // 如果sizeof(BST) != sizeof(BalancedBST)，则会segmentation fault
 
	return 0;
}
```

C++允许你通过基类指针和引用来操作派生类数组。不过这根本就不是一个特性，因为这样的代码几乎从不如你所愿地那样运行。数组与多态不能用在一起。值得注意的是如果你不从一个具体类(concrete classes)(例如BST)派生出另一个具体类(例如BalancedBST)，那么你就不太可能犯这种使用多态性数组的错误。


## 4. 避免无用的缺省构造函数

```
class EquipmentPiece {
public:
	EquipmentPiece(int IDNumber) {}
};
 
int test_item_4()
{
	//EquipmentPiece bestPieces[10]; // 错误，没有正确调用EquipmentPiece构造函数
	//EquipmentPiece* bestPieces2 = new EquipmentPiece[10]; // 错误，与上面的问题一样
 
	int ID1 = 1, ID2 = 2;
	EquipmentPiece bestPieces3[] = { EquipmentPiece(ID1), EquipmentPiece(ID2) }; // 正确，提供了构造函数的参数
 
	// 利用指针数组来代替一个对象数组
	typedef EquipmentPiece* PEP; // PEP指针指向一个EquipmentPiece对象
	PEP bestPieces4[10]; // 正确，没有调用构造函数
	PEP* bestPieces5 = new PEP[10]; // 也正确
	// 在指针数组里的每一个指针被重新赋值，以指向一个不同的EquipmentPiece对象
	for (int i = 0; i < 10; ++i)
		bestPieces5[i] = new EquipmentPiece(ID1);
 
	// 为数组分配raw memory,可以避免浪费内存，使用placement new方法在内存中构造EquipmentPiece对象
	void* rawMemory = operator new[](10*sizeof(EquipmentPiece));
	// make bestPieces6 point to it so it can be treated as an EquipmentPiece array
	EquipmentPiece* bestPieces6 = static_cast<EquipmentPiece*>(rawMemory);
	// construct the EquipmentPiece objects in the memory使用"placement new"
	for (int i = 0; i < 10; ++i)
		new(&bestPieces6[i]) EquipmentPiece(ID1);
	// ...
	// 以与构造bestPieces6对象相反的顺序解构它
	for (int i = 9; i >= 0; --i)
		bestPieces6[i].~EquipmentPiece(); // 如果使用普通的数组删除方法，程序的运行将是不可预测的
	// deallocate the raw memory
	delete [] rawMemory;
 
	return 0;
}
```

构造函数能初始化对象，而缺省构造函数则可以不利用任何在建立对象时的外部数据就能初始化对象。有时这样的方法是不错的。例如一些行为特性与数字相仿的对象被初始化为空值或不确定的值也是合理的，还有比如链表、哈希表、图等等数据结构也可以被初始化为空容器。但不是所有的对象都属于上述类型，**对于很多对象来说，不利用外部数据进行完全的初始化是不合理的**。比如一个没有输入姓名的地址薄对象，就没有任何意义。

利用指针数组代替一个对象数组这种方法有两个缺点：第一你必须删除数组里每个指针所指向的对象。如果忘了，就会发生内存泄漏。第二增加了内存分配量，因为正如你需要空间来容纳EquipmentPiece对象一样，你也需要空间来容纳指针。

对于类里没有定义缺省构造函数还会造成它们无法在许多基于模板(template-based)的容器类里使用。因为实例化一个模板时，模板的类型参数应该提供一个缺省构造函数。在多数情况下，通过仔细设计模板可以杜绝对缺省构造函数的需求。


### 5. 谨慎定义类型转换函数

```
class Name {
public:
	Name(const std::string& s); // 转换string到Name
};
 
class Rational {
public:
	Rational(int numerator = 0, int denominator = 1) // 转换int到有理数类
	{
		n = numerator;
		d = denominator;
	}
 
	operator double() const // 转换Rational类成double类型
	{
		return static_cast<double>(n) / d;
	}
 
	double asDouble() const
	{
		return static_cast<double>(n) / d;
	}
 
private:
	int n, d;
};
 
template<class T>
class Array {
public:
	Array(int lowBound, int highBound) {}
	explicit Array(int size) {}
	T& operator[](int index) { return data[index]; }
 
private:
	T* data;
};
 
bool operator== (const Array<int>& lhs, const Array<int>& rhs)
{ return false; }
 
int test_item_5()
{
	Rational r(1, 2); // r的值是1/2
	double d = 0.5 * r; // 转换r到double,然后做乘法
	fprintf(stdout, "value: %f\n", d);
 
	std::cout<<r<<std::endl; // 应该打印出"1/2",但事与愿违,是一个浮点数，而不是一个有理数,隐式类型转换的缺点
				 // 解决方法是不使用语法关键字的等同的函数来替代转换运算符,如增加asDouble函数，去掉operator double
 
	Array<int> a(10);
	Array<int> b(10);
	for (int i = 0; i < 10; ++i) {
		//if (a == b[i]) {} // 如果构造函数Array(int size)没有explicit关键字，编译器将能通过调用Array<int>构造函数能转换int类型到Array<int>类型，这个构造函数只有一个int类型的参数,加上explicit关键字则可避免隐式转换
 
		if (a == Array<int>(b[i])) {} // 正确，显示从int到Array<int>转换(但是代码的逻辑不合理)
		if (a == static_cast<Array<int>>(b[i]))	 {} // 同样正确，同样不合理
		if (a == (Array<int>)b[i]) {} // C风格的转换也正确，但是逻辑依旧不合理
	}
	return 0;
}
```

C++编译器能够在两种数据类型之间进行隐式转换(implicit conversions)，它继承了C语言的转换方法，例如允许把char隐式转换为int和从short隐式转换为double。你对这些类型转换是无能为力的，因为它们是语言本身的特性。不过当你增加自己的类型时，你就可以有更多的控制力，因为你能选择是否提供函数让编译器进行隐式类型转换。

有两种函数允许编译器进行这些的转换：单参数构造函数(single-argument constructors)和隐式类型转换运算符。单参数构造函数是指只用一个参数即可调用的构造函数。该函数可以是只定义了一个参数，也可以是虽定义了多个参数但第一个参数以后的所有参数都有缺省值。

隐式类型转换运算符只是一个样子奇怪的成员函数：operator关键字，其后跟一个类型符号。你不用定义函数的返回类型，因为返回类型就是这个函数的名字。

explicit关键字是为了解决隐式类型转换而特别引入的这个特性。如果构造函数用explicit声明，编译器会拒绝为了隐式类型转换而调用构造函数。显式类型转换依然合法。

## 6. 自增(increment)、自减(decrement)操作符前缀形式与后缀形式的区别

```
class UPInt { // unlimited precision int
public:
	// 注意：前缀与后缀形式返回值类型是不同的，前缀形式返回一个引用，后缀形式返回一个const类型
	UPInt& operator++() // ++前缀
	{
		//*this += 1; // 增加
		i += 1;
		return *this; // 取回值
	}
 
	const UPInt operator++(int) // ++后缀
	{
		// 注意：建立了一个显示的临时对象，这个临时对象必须被构造并在最后被析构，前缀没有这样的临时对象
		UPInt oldValue = *this; // 取回值
		// 后缀应该根据它们的前缀形式来实现
		++(*this); // 增加
		return oldValue; // 返回被取回的值
	}
 
	UPInt& operator--() // --前缀
	{
		i -= 1;
		return *this;
	}
 
	const UPInt operator--(int) // --后缀
	{
		UPInt oldValue = *this;
		--(*this);
		return oldValue;
	}
 
	UPInt& operator+=(int a) // +=操作符，UPInt与int相运算
	{
		i += a;
		return *this;
	}
 
	UPInt& operator-=(int a)
	{
		i -= a;
		return *this;
	}
 
private:
	int i;
}; 
 
int test_item_6()
{
	UPInt i;
	++i; // 调用i.operator++();
	i++; // 调用i.operator++(0);
	--i; // 调用i.operator--();
	i--; // 调用i.operator--(0);
 
	//i++++; // 注意：++后缀返回的是const UPInt
 
	return 0;
}
```

无论是increment或decrement的前缀还是后缀都只有一个参数，为了解决这个语言问题，C++规定后缀形式有一个int类型参数，当函数被调用时，编译器传递一个0作为int参数的值给该函数。

前缀形式有时叫做”增加然后取回”，后缀形式叫做”取回然后增加”。

当处理用户定义的类型时，尽可能地使用前缀increment，因为它的效率较高。


### 7. 不要重载”&&”, “||”,或”,”

```
int test_item_7()
{
	// if (expression1 && expression2)
	// 如果重载了操作符&&，对于编译器来说，等同于下面代码之一
	// if (expression1.operator&&(expression2)) // when operator&& is a member function
	// if (operator&&(expression1, expression2)) // when operator&& is a global function
 
	return 0;
}
```

与C一样，C++使用布尔表达式短路求值法(short-circuit evaluation)。这表示一旦确定了布尔表达式的真假值，即使还有部分表达式没有被测试，布尔表达式也停止运算。

C++允许根据用户定义的类型，来定制&&和||操作符。方法是重载函数operator&&和operator||，你能在全局重载或每个类里重载。**风险：你以函数调用法替代了短路求值法**。函数调用法与短路求值法是绝对不同的。首先当函数被调用时，需要运算其所有参数。第二是C++语言规范没有定义函数参数的计算顺序，所以没有办法知道表达式1与表达式2哪一个先计算。完全可能与具有从左参数到右参数计算顺序的短路计算法相反。因此如果你重载&&或||，就没有办法提供给程序员他们所期望和使用的行为特性，所以不要重载&&和||。

同样的理由也适用于逗号操作符。逗号操作符用于组成表达式。一个包含逗号的表达式首先计算逗号左边的表达式，然后计算逗号右边的表达式；整个表达式的结果是逗号右边表达式的值。如果你写一个非成员函数operator，你不能保证左边的表达式先于右边的表达式计算，因为函数(operator)调用时两个表达式作为参数被传递出去。但是你不能控制函数参数的计算顺序。所以非成员函数的方法绝对不行。成员函数operator，你也不能依靠于逗号左边表达式先被计算的行为特性，因为编译器不一定必须按此方法去计算。因此你不能重载逗号操作符，保证它的行为特性与其被料想的一样。重载它是完全轻率的行为。


## 8. 理解各种不同含义的new和delete

```
class Widget8 {
public:
	Widget8(int widget8Size) {}
};
 
void* mallocShared(size_t size)
{
	return operator new(size);
}
 
void freeShared(void* memory)
{
	operator delete(memory);
}
 
Widget8* constructWidget8InBuffer(void* buffer, int widget8Size)
{
	return new(buffer) Widget8(widget8Size); // new操作符的一个用法，需要使用一个额外的变量(buffer)，当new操作符隐含调用operator new函数时，把这个变量传递给它
	// 被调用的operator new函数除了待有强制的参数size_t外，还必须接受void*指针参数，指向构造对象占用的内存空间。这个operator new就是placement new,它看上去像这样:
	// void * operator new(size_t, void* location) { return location; }
}
 
int test_item_8()
{
	std::string* ps = new std::string("Memory Management"); // 使用的new是new操作符(new operator)
	//void * operator new(size_t size); // 函数operator new通常声明
	void* rawMemory = operator new(sizeof(std::string)); // 操作符operator new将返回一个指针，指向一块足够容纳一个string类型对象的内存
	operator delete(rawMemory);
 
	delete ps; // ps->~std::string(); operator delete(ps);
 
	void* buffer = operator new(50*sizeof(char)); // 分配足够的内存以容纳50个char，没有调用构造函数
	operator delete(buffer); // 释放内存，没有调用析构函数. 这与在C中调用malloc和free等同OA
 
	void* sharedMemory = mallocShared(sizeof(Widget8));
	Widget8* pw = constructWidget8InBuffer(sharedMemory, 10); // placement new
	//delete pw; // 结果不确定，共享内存来自mallocShared,而不是operator new
	pw->~Widget8(); // 正确，析构pw指向的Widget8,但是没有释放包含Widget8的内存
	freeShared(pw); // 正确，释放pw指向的共享内存，但是没有调用析构函数
 
	return 0;
}
```

new操作符(new operator)和new操作(operator new)的区别：

new操作符就像sizeof一样是语言内置的，你不能改变它的含义，它的功能总是一样的。它要完成的功能分成两部分。第一部分是分配足够的内存以便容纳所需类型的对象。第二部分是它调用构造函数初始化内存中的对象。new操作符总是做这两件事情，你不能以任何方式改变它的行为。你所能改变的是如何为对象分配内存。new操作符调用一个函数来完成必须的内存分配，你能够重写或重载这个函数来改变它的行为。new操作符为分配内存所调用函数的名字是operator new。

函数operator new通常声明：返回值类型是void*，因为这个函数返回一个未经处理(raw)的指针，未初始化的内存。参数size_t确定分配多少内存。你能增加额外的参数重载函数operator new，但是第一个参数类型必须是size_t。就像malloc一样，operator new的职责只是分配内存。它对构造函数一无所知。把operator new返回的未经处理的指针传递给一个对象是new操作符的工作。

placement new：特殊的operator new，接受的参数除了size_t外还有其它。

new操作符(new operator)与operator new关系：你想在堆上建立一个对象，应该用new操作符。它既分配内存又为对象调用构造函数。如果你仅仅想分配内存，就应该调用operator new函数，它不会调用构造函数。如果你想定制自己的在堆对象被建立时的内存分配过程，你应该写你自己的operator new函数，然后使用new操作符，new操作符会调用你定制的operator new。如果你想在一块已经获得指针的内存里建立一个对象，应该用placement new。

Deletion and Memory Deallocation：为了避免内存泄漏，每个动态内存分配必须与一个等同相反的deallocation对应。函数operator delete与delete操作符的关系与operator new与new操作符的关系一样。

如果你用placement new在内存中建立对象，你应该避免在该内存中用delete操作符。因为delete操作符调用operator delete来释放内存，但是包含对象的内存最初不是被operator nen分配的，placement new只是返回转到给它的指针。

Arrays：operator new[]、operator delete[]


## 9. 使用析构函数防止资源泄漏

用一个对象存储需要被自动释放的资源，然后依靠对象的析构函数来释放资源，这种思想不只是可以运用在指针上，还能用在其它资源的分配和释放上。

资源应该被封装在一个对象里，遵循这个规则，你通常就能够避免在存在异常环境里发生资源泄漏，通过智能指针的方式。

C++确保删除空指针是安全的，所以析构函数在删除指针前不需要检测这些指针是否指向了某些对象。


## 10. 在构造函数中防止资源泄漏

C++仅仅能删除被完全构造的对象(fully constructed objects)，只有一个对象的构造函数完全运行完毕，这个对象才被完全地构造。C++拒绝为没有完成构造操作的对象调用析构函数。

在构造函数中可以使用try catch throw捕获所有的异常。更好的解决方法是通过智能指针的方式。

如果你用对应的std::unique_ptr对象替代指针成员变量，就可以防止构造函数在存在异常时发生资源泄漏，你也不用手工在析构函数中释放资源，并且你还能像以前使用非const指针一样使用const指针，给其赋值。

std::unique_ptr的使用参考：https://blog.csdn.net/fengbingchun/article/details/52203664


## 11. 禁止异常信息(exceptions)传递到析构函数外

禁止异常传递到析构函数外有两个原因：第一能够在异常传递的堆栈辗转开解(stack-unwinding)的过程中，防止terminate被调用。第二它能帮助确保析构函数总能完成我们希望它做的所有事情。



## 12. 理解”抛出一个异常”与”传递一个参数”或”调用一个虚函数”间的差异

你调用函数时，程序的控制权最终还会返回到函数的调用处，但是当你抛出一个异常时，控制权永远不会回到抛出异常的地方。

C++规范要求被作为异常抛出的对象必须被复制。所以一旦控制权离开了，局部变量的析构函数就会被调用。即使被抛出的对象不会被释放，也会进行拷贝操作。抛出异常运行速度比参数传递要慢。

当异常对象被拷贝时，拷贝操作是由对象的拷贝构造函数完成的。该拷贝构造函数是对象的静态类型(static type)所对应类的拷贝构造函数，而不是对象的动态类型(dynamic type)对应类的拷贝构造函数。

by-value方式的异常捕获，会发生两次复制，而通过引用或常引用则只会复制一次。

catch子句中进行异常匹配时可以进行两种类型转换：第一种是继承类与基类间的转换。一个用来捕获基类的catch子句也可以处理派生类类型的异常。这种派生类与基类(inheritance_based)间的异常类型转换可以作用于数值、引用以及指针上。第二种是允许从一个类型化指针(typed pointer)转变成无类型指针(untyped pointer)，所以带有const void*指针的catch子句能捕获任何类型的指针类型异常。

catch子句匹配顺序总是取决于它们在程序中出现的顺序。因此一个派生类异常可能被处理其基类异常的catch子句捕获，即使同时存在有能直接处理该派生类异常的catch子句，与相同的try块相对应。不要把处理基类异常的catch子句放在处理派生类异常的catch子句的前面。

把一个对象传递给函数或一个对象调用虚拟函数与把一个对象作为异常抛出，这之间有三个主要区别：第一，异常对象在传递时总被进行拷贝；当通过传值方式捕获时，异常对象被拷贝了两次。对象作为参数传递给函数时不一定需要被拷贝。第二，对象作为异常被抛出与作为参数传递给函数相比，前者类型转换比后者要少(前者只有两种转换形式)。最后一点，catch子句进行异常类型匹配的顺序是它们在源代码中出现的顺序，第一个类型匹配成功的catch将被用来执行。当一个对象调用一个虚拟函数时，被选择的函数位于与对象类型匹配最佳的类里，即使该类不是在源代码的最前头。

try catch介绍参考：https://blog.csdn.net/fengbingchun/article/details/65939258


## 13. 通过引用(reference)捕获异常

by-value方式的缺点：by-value方式的异常捕获，会发生两次复制。同时derived class对象传递给base class对象，会发生切割。

通过指针捕获异常不符合C++语言本身的规范。四个标准的异常----bad_alloc(当operator new不能分配足够的内存时被抛出)；bad_cast(当dynamic_cast针对一个引用(reference)操作失败时被抛出)；bad_typeid(当dynamic_cast对空指针进行操作时被抛出)；bad_exception(用于unexpected异常)----都不是指向对象的指针，所以你必须通过值或引用来捕获它们。

std::exception的介绍参考：https://blog.csdn.net/fengbingchun/article/details/78303734


## 14. 审慎使用异常规格(exception specifications)

如果一个函数抛出一个不在异常规格范围里的异常，系统在运行时能够检测出这个错误，然后一个特殊函数std::unexpected将被自动地调用(This function is automatically called when a function throws an exception that is not listed in its dynamic-exception-specifier.)。std::unexpected缺省的行为是调用函数std::terminate，而std::terminate缺省的行为是调用函数abort。应避免调用std::unexpected。

避免在带有类型参数的模板内使用异常规格。

C++允许你用其它不同的异常类型替换std::unexpected异常，通过std::set_unexpected。


## 15. 了解异常处理的系统开销

采用不支持异常的方法编译的程序一般比支持异常的程序运行速度更快所占空间也更小。

为了减少开销，你应该避免使用无用的try块。如果使用try块，代码的尺寸将增加并且运行速度也会减慢。



## 16. 牢记80-20准则(80-20 rule)

80-20准则说的是大约20%的代码使用了80%的程序资源；大约20%的代码耗用了大约80%的运行时间；大约20%的代码使用了80%的内存；大约20%的代码执行80%的磁盘访问；80%的维护投入于大约20%的代码上。基本的观点：软件整体的性能取决于代码组成中的一小部分。



## 17. 考虑使用lazy evaluation(懒惰计算法)

在某些情况下要求软件进行原来可以避免的计算，这时lazy evaluation才是有用的。



## 18. 分期摊还期望的计算

over-eager evaluation(过度热情计算法)：在要求你做某些事情以前就完成它们。隐藏在over-eager evaluation后面的思想是如果你认为一个计算需要频繁进行，你就可以设计一个数据结构高效地处理这些计算需求，这样可以降低每次计算需求时的开销。

当你必须支持某些操作而不总需要其结果时，lazy evaluation是在这种时候使用的用以提高程序效率的技术。当你必须支持某些操作而其结果几乎总是被需要或不止一次地需要时，over-eager是在这种时候使用的用以提高程序效率的一种技术。


## 19. 理解临时对象的来源

```
size_t countChar(const std::string& str, char ch)
{
	// 建立一个string类型的临时对象，通过以buffer做为参数调用string的构造函数来初始化这个临时对象,
	// countChar的参数str被绑定在这个临时的string对象上，当countChar返回时，临时对象自动释放
 
	// 将countChar(const std::string& str, char ch)修改为countChar(std::string& str, char ch)则会error
	return 1;
}
 
#define MAX_STRING_LEN 64
 
int test_item_19()
{
	char buffer[MAX_STRING_LEN];
	char c;
 
	std::cin >> c >> std::setw(MAX_STRING_LEN) >> buffer;
	std::cout<<"There are "<<countChar(buffer, c)<<" occurrences of the character "<<c<<" in "<<buffer<<std::endl;
 
	return 0;
}
```

在C++中真正的临时对象是看不见的，它们不出现在你的源代码中。建立一个没有命名的非堆(non-heap)对象会产生临时对象。这种未命名的对象通常在两种条件下产生：为了使函数成功调用而进行隐式类型转换和函数返回对象时。

仅当通过传值(by value)方式传递对象或传递常量引用(reference-to-const)参数时，才会发生这些类型转换。当传递一个非常量引用(reference-to-non-const)参数对象，就不会发生。

C++语言禁止为非常量引用(reference-to-non-const)产生临时对象。

临时对象是有开销的，所以你应该尽可能地去除它们。在任何时候只要见到常量引用(reference-to-const)参数，就存在建立临时对象而绑定在参数上的可能性。在任何时候只要见到函数返回对象，就会有一个临时对象被建立(以后被释放)。


## 20. 协助完成返回值优化

```
class Rational20 {
public:
	Rational20(int numerator = 0, int denominator = 1) {}
 
	int numerator() const { return 1; }
	int denominator() const { return 2; }
};
 
const Rational20 operator*(const Rational20& lhs, const Rational20& rhs)
{
	// 以某种方法返回对象，能让编译器消除临时对象的开销：这种技巧是返回constructor argument而不是直接返回对象
	return Rational20(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
 
int test_item_20()
{
	Rational20 a = 10;
	Rational20 b(1, 2);
	Rational20 c = a * b; 
 
	return 0;
}
```

一些函数(operator*也在其中)必须要返回对象。这就是它们的运行方法。

C++规则允许编译器优化不出现的临时对象(temporary objects out of existence)



## 21. 通过重载避免隐式类型转换

```
class UPInt21 { // unlimited precision integers class
public:
	UPInt21() {}
	UPInt21(int value) {}
};
 
const UPInt21 operator+(const UPInt21& lhs, const UPInt21& rhs) // add UPInt21+UPInt21
{
	return UPInt21(1);
}
 
const UPInt21 operator+(const UPInt21& lhs, int rhs) // add UPInt21+int
{
	return UPInt21(1);
}
 
const UPInt21 operator+(int lhs, const UPInt21& rhs) // add int+UPInt21
{
	return UPInt21(1);
}
 
int test_item_21()
{
	UPInt21 upi1, upi2;
	UPInt21 upi3 = upi1 + upi2; // 正确，没有由upi1或upi2生成临时对象
	upi3 = upi1 + 10; // 正确,没有由upi1或10生成临时对象
	upi3 = 10 + upi2; // 正确，没有由10或upi2生成临时对象
 
	// 注意：注释掉上面的operator+(UPInt21&, int)和operator+(int, UPInt21&)也正确，但是会通过临时对象把10转换为UPInt21
 
	return 0;
}
```

在C++中有一条规则是每一个重载的operator必须带有一个用户定义类型(user-defined type)的参数。

利用重载避免临时对象的方法不只是用在operator函数上。

没有必要实现大量的重载函数，除非你有理由确信程序使用重载函数以后其整体效率会有显著的提高。



## 22. 考虑用运算符的赋值形式(op=)取代其单独形式(op)

```
class Rational22 {
public:
	Rational22(int numerator = 0, int denominator = 1) {}
	Rational22& operator+=(const Rational22& rhs) { return *this; }
	Rational22& operator-=(const Rational22& rhs) { return *this; }
};
 
// operator+根据operator+=实现
const Rational22 operator+(const Rational22& lhs, const Rational22& rhs)
{
	return Rational22(lhs) += rhs;
}
 
// operator-根据operator-=实现
const Rational22 operator-(const Rational22& lhs, const Rational22& rhs)
{
	return Rational22(lhs) -= rhs;
}
```

就C++来说，operator+、operator=和operator+=之间没有任何关系，因此如果你想让三个operator同时存在并具有你所期望的关系，就必须自己实现它们。同理，operator-, *, /, 等等也一样。

确保operator的赋值形式(assignment version)(例如operator+=)与一个operator的单独形式(stand-alone)(例如operator+)之间存在正常的关系，一种好方法是后者(指operator+)根据前者(指operator+=)来实现。



## 23. 考虑变更程序库

不同的程序库在效率、可扩展性、移植性、类型安全和其它一些领域上蕴含着不同的设计理念，通过变换使用给予性能更多考虑的程序库，你有时可以大幅度地提供软件的效率。



## 24. 理解虚拟函数、多继承、虚基类和RTTI所需的代码

当调用一个虚拟函数时，被执行的代码必须与调用函数的对象的动态类型相一致；指向对象的指针或引用的类型是不重要的。大多数编译器是使用virtual table和virtual table pointers，通常被分别地称为vtbl和vptr。

一个vtbl通常是一个函数指针数组。(一些编译器使用链表来代替数组，但是基本方法是一样的)在程序中的每个类只要声明了虚函数或继承了虚函数，它就有自己的vtbl，并且类中vtbl的项目是指向虚函数实现体的指针。

你必须为每个包含虚函数的类的virtual table留出空间。类的vtbl的大小与类中声明的虚函数的数量成正比(包括从基类继承的虚函数)。每个类应该只有一个virtual table，所以virtual table所需的空间不会太大，但是如果你有大量的类或者在每个类中有大量的虚函数，你会发现vtbl会占用大量的地址空间。

一些原因导致现在的编译器一般总是忽略虚函数的inline指令。

Virtual table只实现了虚拟函数的一半机制，如果只有这些是没有用的。只有用某种方法指出每个对象对应的vtbl时，它们才能使用。这是virtual table pointer的工作，它来建立这种联系。每个声明了虚函数的对象都带着它，它是一个看不见的数据成员，指向对应类的virtual table。这个看不见的数据成员也称为vptr，被编译器加在对象里，位置只有编译器知道。

关于虚函数表的介绍参考：https://blog.csdn.net/fengbingchun/article/details/79592347

虚函数是不能内联的。这是因为”内联”是指”在编译期间用被调用的函数体本身来代替函数调用的指令”，但是虚函数的”虚”是指”直到运行时才能知道要调用的是哪一个函数”。

RTTI(运行时类型识别)能让我们在运行时找到对象和类的有关信息，所以肯定有某个地方存储了这些信息让我们查询。这些信息被存储在类型为type_info的对象里，你能通过使用typeid操作符访问一个类的type_info对象。

关于typeid的使用参考：https://blog.csdn.net/fengbingchun/article/details/51866559

RTTI被设计为在类的vtbl基础上实现。



## 25. 将构造函数和非成员函数虚拟化

虚拟构造函数是指能够根据输入给它的数据的不同而建立不同类型的对象。虚拟拷贝构造函数能返回一个指针，指向调用该函数的对象的新拷贝。类的虚拟拷贝构造函数只是调用它们真正的拷贝构造函数。被派生类重定义的虚拟函数不用必须与基类的虚拟函数具有一样的返回类型。如果函数的返回类型是一个指向基类的指针(或一个引用)，那么派生类的函数可以返回一个指向基类的派生类的指针(或引用)。



## 26. 限制某个类所能产生的对象数量

阻止建立某个类的对象，最容易的方法就是把该类的构造函数声明在类的private域。



## 27. 要求或禁止在堆中产生对象

```
// 判断一个对象是否在堆中, HeapTracked不能用于内建类型，因为内建类型没有this指针
typedef const void* RawAddress;
class HeapTracked { // 混合类，跟踪
public:
	class MissingAddress {}; // 从operator new返回的ptr异常类
	virtual ~HeapTracked() = 0;
	static void* operator new(size_t size);
	static void operator delete(void* ptr);
	bool isOnHeap() const;
 
private:
	static std::list<RawAddress> addresses;
};
 
std::list<RawAddress> HeapTracked::addresses;
 
HeapTracked::~HeapTracked() {}
 
void* HeapTracked::operator new(size_t size)
{
	void* memPtr = ::operator new(size);
	addresses.push_front(memPtr);
	return memPtr;
}
 
void HeapTracked::operator delete(void* ptr)
{
	std::list<RawAddress>::iterator it = std::find(addresses.begin(), addresses.end(), ptr);
	if (it != addresses.end()) {
		addresses.erase(it);
		::operator delete(ptr);
	} else {
		throw MissingAddress(); // ptr就不是用operator new分配的，所以抛出一个异常
	}
}
 
bool HeapTracked::isOnHeap() const
{
	// 生成的指针将指向"原指针指向对象内存"的开始处
	// 如果HeapTracked::operator new为当前对象分配内存，这个指针就是HeapTracked::operator new返回的指针
	const void* rawAddress = dynamic_cast<const void*>(this);
	std::list<RawAddress>::iterator it = std::find(addresses.begin(), addresses.end(), rawAddress);
	return it != addresses.end();
}
 
class Asset : public HeapTracked {};
 
// 禁止堆对象
class UPNumber27 {
private:
	static void* operator new(size_t size);
	static void operator delete(void* ptr);
};
 
void* UPNumber27::operator new(size_t size)
{
	return ::operator new(size);
}
 
void UPNumber27::operator delete(void* ptr)
{
	::operator delete(ptr);
}
 
class Asset27 {
public:
	Asset27(int initValue) {}
 
private:
	UPNumber27 value;
};
 
int test_item_27()
{
	UPNumber27 n1; // okay
	static UPNumber27 n2; // also okay
	//UPNumber27* p = new UPNumber27; // error, attempt to call private operator new
 
	// UPNumber27的operator new是private这一点 不会对包含UPNumber27成员对象的对象的分配产生任何影响
	Asset27* pa = new Asset27(100); // 正确，调用Asset::operator new或::operator new,不是UPNumber27::operator new
 
	return 0;
}
```

禁止堆对象：禁止用于调用new，利用new操作符总是调用operator new函数这点来达到目的，可以自己声明这个函数，而且你可以把它声明为private。



## 28. smart指针

```
template<class T>
class SmartPtr {
public:
	SmartPtr(T* realPtr = 0); // 建立一个灵巧指针指向dumb pointer(内建指针)所指的对象，未初始化的指针，缺省值为0(null)
	SmartPtr(const SmartPtr& rhs); // 拷贝一个灵巧指针
	~SmartPtr(); // 释放灵巧指针
	// make an assignment to a smart ptr
	SmartPtr& operator=(const SmartPtr& rhs);
	T* operator->() const; // dereference一个灵巧指针以访问所指对象的成员
	T& operator*() const; // dereference灵巧指针
 
private:
	T* pointee; // 灵巧指针所指的对象
};
```

灵巧指针是一种外观和行为都被设计成与内建指针相类似的对象，不过它能提供更多的功能。它们有许多应用的领域，包括资源管理和重复代码任务的自动化。

在C++11中auto_ptr已经被废弃，用unique_ptr替代。

std::unique_ptr的使用参考：https://blog.csdn.net/fengbingchun/article/details/52203664



## 29. 引用计数

```
class String {
public:
	String(const char* initValue = "");
	String(const String& rhs);
	String& operator=(const String& rhs);
	const char& operator[](int index) const; // for const String
	char& operator[](int index); // for non-const String
	~String();
 
private:
	// StringValue的主要目的是提供一个空间将一个特别的值和共享此值的对象的数目联系起来
	struct StringValue { // holds a reference count and a string value
		int refCount;
		char* data;
		bool shareable; // 标志，以指出它是否为可共享的
		StringValue(const char* initValue);
		~StringValue();
	};
 
	StringValue* value; // value of this String
};
 
String::String(const char* initValue) : value(new StringValue(initValue))
{}
 
String::String(const String& rhs)
{
	if (rhs.value->shareable) {
		value = rhs.value;
		++value->refCount;
	} else {
		value = new StringValue(rhs.value->data);
	}
}
 
String& String::operator=(const String& rhs)
{
	if (value == rhs.value) { // do nothing if the values are already the same
		return *this;
	}
 
	if (value->shareable && --value->refCount == 0) { // destroy *this's value if no one else is using it
		delete value;
	}
 
	if (rhs.value->shareable) {
		value = rhs.value; // have *this share rhs's value
		++value->refCount;
	} else {
		value = new StringValue(rhs.value->data);
	}
 
	return *this;
}
 
const char& String::operator[](int index) const
{
	return value->data[index];
}
 
char& String::operator[](int index)
{
	// if we're sharing a value with other String objects, break off a separate copy of the value fro ourselves
	if (value->refCount > 1) {
		--value->refCount; // decrement current value's refCount, becuase we won't be using that value any more
		value = new StringValue(value->data); // make a copy of the value for ourselves
	}
 
	value->shareable = false;
	// return a reference to a character inside our unshared StringValue object
	return value->data[index];
}
 
String::~String()
{
	if (--value->refCount == 0) {
		delete value;
	}
}
 
String::StringValue::StringValue(const char* initValue) : refCount(1), shareable(true)
{
	data = new char[strlen(initValue) + 1];
	strcpy(data, initValue);
}
 
String::StringValue::~StringValue()
{
	delete[] data;
}
 
// 基类，任何需要引用计数的类都必须从它继承
class RCObject {
public:
	void addReference() { ++refCount; }
	void removeReference() { if (--refCount == 0) delete this; } // 必须确保RCObject只能被构建在堆中
	void markUnshareable() { shareable = false; }
	bool isShareable() const { return shareable; }
	bool isShared() const { return refCount > 1; }
 
protected:
	RCObject() : refCount(0), shareable(true) {}
	RCObject(const RCObject& rhs) : refCount(0), shareable(true) {}
	RCObject& operator=(const RCObject& rhs) { return *this; }
	virtual ~RCObject() = 0;
 
private:
	int refCount;
	bool shareable;
 
};
 
RCObject::~RCObject() {} // virtual dtors must always be implemented, even if they are pure virtual and do nothing
 
// template class for smart pointers-to-T objects. T must support the RCObject interface, typically by inheriting from RCObject
template<class T>
class RCPtr {
public:
	RCPtr(T* realPtr = 0) : pointee(realPtr) { init(); }
	RCPtr(const RCPtr& rhs) : pointee(rhs.pointee) { init(); }
	~RCPtr() { if (pointee) pointee->removeReference(); }
 
	RCPtr& operator=(const RCPtr& rhs)
	{
		if (pointee != rhs.pointee) { // skip assignments where the value doesn't change
			if (pointee)
				pointee->removeReference(); // remove reference to current value
 
			pointee = rhs.pointee; // point to new value
			init(); // if possible, share it else make own copy
		}
 
		return *this;
	}
 
	T* operator->() const { return pointee; }
	T& operator*() const { return *pointee; }
 
private:
	T* pointee; // dumb pointer this object is emulating
 
	void init() // common initialization
	{
		if (pointee == 0) // if the dumb pointer is null, so is the smart one
			return;
 
		if (pointee->isShareable() == false) // if the value isn't shareable copy it
			pointee = new T(*pointee);
 
		pointee->addReference(); // note that there is now a new reference to the value
	}
};
 
// 将StringValue修改为是从RCObject继承
// 将引用计数功能移入一个新类(RCObject)，增加了灵巧指针(RCPtr)来自动处理引用计数
class String2 {
public:
	String2(const char* value = "") : value(new StringValue(value)) {}
	const char& operator[](int index) const { return value->data[index]; } // for const String2
	
	char& operator[](int index) // for non-const String2
	{
		if (value->isShared())
			value = new StringValue(value->data);
		value->markUnshareable();
		return value->data[index];
	}
 
private:
	// StringValue的主要目的是提供一个空间将一个特别的值和共享此值的对象的数目联系起来
	struct StringValue : public RCObject { // holds a reference count and a string value
		char* data;
 
		StringValue(const char* initValue) { init(initValue); }
		StringValue(const StringValue& rhs) { init(rhs.data); }
 
		void init(const char* initValue)
		{
			data = new char[strlen(initValue) + 1];
			strcpy(data, initValue);
		}
 
		~StringValue() { delete [] data; }
	};
 
	RCPtr<StringValue> value; // value of this String2
 
};
 
int test_item_29()
{
	String s1("More Effective C++");
	String s2 = s1;
	s1 = s2;
	fprintf(stdout, "char: %c\n", s1[2]);
	String s3 = s1;
	s3[5] = 'x';
 
	return 0;
}
```

引用计数是这样一个技巧，它允许多个有相同值的对象共享这个值的实现。这个技巧有两个常用动机。第一个是简化跟踪堆中的对象的过程。一旦一个对象通过调用new被分配出来，最要紧的就是记录谁拥有这个对象，因为其所有者----并且只有其所有者----负责对这个对象调用delete。但是，所有权可以被从一个对象传递到另外一个对象(例如通过传递指针型参数)。引用计数可以免除跟踪对象所有权的担子，因为当使用引用计数后，对象自己拥有自己。当没人再使用它时，它自己自动销毁自己。因此，引用计数是个简单的垃圾回收体系。第二个动机是由于一个简单的常识。如果很多对象有相同的值，将这个值存储多次是很无聊的。更好的办法是让所有的对象共享这个值的实现。这么做不但节省内存，而且可以使得程序运行更快，因为不需要构造和析构这个值的拷贝。

引用计数介绍参考：https://blog.csdn.net/fengbingchun/article/details/85861776

实现引用计数不是没有代价的。每个被引用的值带一个引用计数，其大部分操作都需要以某种形式检查或操作引用计数。对象的值需要更多的内存，而我们在处理它们时需要执行更多的代码。引用计数是基于对象通常共享相同的值的假设的优化技巧。如果假设不成立的话，引用计数将比通常的方法使用更多的内存和执行更多的代码。另一方面，如果你的对象确实有具有相同值的趋势，那么引用计数将同时节省时间和空间。



## 30. 代理类

```
template<class T>
class Array2D { // 使用代理实现二维数组
public:
	Array2D(int i, int j) : i(i), j(j)
	{
		data.reset(new T[i*j]);
	}
 
	class Array1D { // Array1D是一个代理类，它的实例扮演的是一个在概念上不存在的一维数组
	public:
		Array1D(T* data) : data(data) {}
		T& operator[](int index) { return data[index]; }
		const T& operator[](int index) const { return data[index]; }
 
	private:
		T* data;
	};
 
	Array1D operator[](int index) { return Array1D(data.get()+j*index); }
	const Array1D operator[](int index) const { return Array1D(data.get()+j*index); }
 
private:
	std::unique_ptr<T[]> data;
	int i, j;
};
 
// 可以通过代理类帮助区分通过operator[]进行的是读操作还是写操作
class String30 {
public:
	String30(const char* value = "") : value(new StringValue(value)) {}
	
	class CharProxy { // proxies for string chars
	public:
		CharProxy(String30& str, int index) : theString(str), charIndex(index) {}
 
		CharProxy& operator=(const CharProxy& rhs)
		{
			// if the string is haring a value with other String objects,
			// break off a separate copy of the value for this string only
			if (theString.value->isShared())
				theString.value = new StringValue(theString.value->data);
 
			// now make the assignment: assign the value of the char
			// represented by rhs to the char represented by *this
			theString.value->data[charIndex] = rhs.theString.value->data[rhs.charIndex];
			return *this;
		}
		
		CharProxy& operator=(char c)
		{
			if (theString.value->isShared())
				theString.value = new StringValue(theString.value->data);
			theString.value->data[charIndex] = c;
			return *this;
		}
 
		operator char() const { return theString.value->data[charIndex]; }
 
	private:
		String30& theString;
		int charIndex;
	};
 
	const CharProxy operator[](int index) const // for const String30
	{
		return CharProxy(const_cast<String30&>(*this), index);
	}
 
	CharProxy operator[](int index) // for non-const String30
	{
		return CharProxy(*this, index);
	}
 
	//friend class CharProxy;
private:
	// StringValue的主要目的是提供一个空间将一个特别的值和共享此值的对象的数目联系起来
	struct StringValue : public RCObject { // holds a reference count and a string value
		char* data;
 
		StringValue(const char* initValue) { init(initValue); }
		StringValue(const StringValue& rhs) { init(rhs.data); }
 
		void init(const char* initValue)
		{
			data = new char[strlen(initValue) + 1];
			strcpy(data, initValue);
		}
 
		~StringValue() { delete [] data; }
	};
 
	RCPtr<StringValue> value; // value of this String30
 
};
 
int test_item_30()
{
	Array2D<float> data(10, 20);
	fprintf(stdout, "%f\n", data[3][6]);
 
	String30 s1("Effective C++"), s2("More Effective C++"); // reference-counted strings using proxies
	fprintf(stdout, "%c\n", s1[5]); // still legal, still works
	s2[5] = 'x'; // also legal, also works
	s1[3] = s2[8]; // of course it's legal, of course it works
 
	//char* p = &s1[1]; // error, 通常,取proxy对象地址的操作与取实际对象地址的操作得到的指针，其类型是不同的,重载CharProxy类的取地址运算可消除这个不同
 
	return 0;
}
```

可以通过代理类实现二维数组。

可以通过代理类帮助区分通过operator[]进行的是读操作还是写操作。

Proxy类可以完成一些其它方法很难甚至可不能实现的行为。多维数组是一个例子，左值/右值的区分是第二个，限制隐式类型转换是第三个。

同时，proxy类也有缺点。作为函数返回值，proxy对象是临时对象，它们必须被构造和析构。Proxy对象的存在增加了软件的复杂度。从一个处理实际对象的类改换到处理proxy对象的类经常改变了类的语义，因为proxy对象通常表现出的行为与实际对象有些微妙的区别。




## 31. 让函数根据一个以上的对象来决定怎么虚拟



## 32. 在未来时态下开发程序

未来时态的考虑增加了你的代码的可重用性、可维护性、健壮性，以及在环境发生改变时易于修改。

## 33. 将非尾端类设计为抽象类



## 34. 如何在同一程序中混合使用C++和C

名变换：就是C++编译器给程序的每个函数换一个独一无二的名字。在C中，这个过程是不需要的，因为没有函数重载，但几乎所有C++程序都有函数重名。要禁止名变换，使用C++的extern “C”。不要将extern “C”看作是声明这个函数是用C语言写的，应该看作是声明这个函数应该被当作好像C写的一样而进行调用。

静态初始化：在main执行前和执行后都有大量代码被执行。尤其是，静态的类对象和定义在全局的、命名空间中的或文件体中的类对象的构造函数通常在main被执行前就被调用。这个过程称为静态初始化。同样，通过静态初始化产生的对象也要在静态析构过程中调用其析构函数，这个过程通常发生在main结束运行之后。

动态内存分配：C++部分使用new和delete，C部分使用malloc(或其变形)和free。

数据结构的兼容性：在C++和C之间这样相互传递数据结构是安全的----在C++和C下提供同样的定义来进行编译。在C++版本中增加非虚成员函数或许不影响兼容性，但几乎其它的改变都将影响兼容。

如果想在同一程序下混合C++与C编程，记住下面的指导原则：(1).确保C++和C编译器产生兼容的obj文件；(2).将在两种语言下都使用的函数声明为extern “C”；(3).只要可能，用C++写main()；(4).总用delete释放new分配的内存；总用free释放malloc分配的内存；(5).将在两种语言间传递的东西限制在用C编译的数据结构的范围内；这些结构的C++版本可以包含非虚成员函数。
