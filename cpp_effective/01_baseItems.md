# 1. 习惯C++

## 序

**声明式**：告诉编译器**名称和类型**，忽略细节：

```
extern int x;
std::size_t numDigits(int number);   // 函数声明式
class Widget;                        // 类声明式

template<typename T>                 // 类模板声明
class GraphNode;                     // typename 使用见42
```

函数的声明指定函数签名，包括参数和返回类型，如numDigits函数的签名是`std::size_t (int)`。

**定义式**：提供给编译器在声明式遗漏的细节，编译器按定义式分配内存。function和function template的定义给定函数本体；class和class template给定成员。

**初始化**：给对象赋初值，由构造函数执行。

默认构造函数：不带任何实参，或每个实参有默认值。

**explicit**：阻止编译器的隐式类型转换行为，仍可以进行显示类型转换。

```
class B{
public:
	explicit B(int x=0, bool b=true); // default构造函数
}
```



```
void doSomething(B bObj);

B bObj1;                 // 正确，其参数有默认值
doSomething(bObj1);      // 正确
B bObj2(28);             // 正确，其参数有默认值

doSomething(28);         // 错误，没有隐式转换
doSomething(B(28));      // 正确
```

所以explicit是更正确的做法。

**copy构造函数**：以同类型对象初始化自我对象。

**copy assignment操作符**：从另一个对象拷贝其值到自我对象。

```
class Widget{
public:
	Widget();
	Widget(const Widget& rhs);
	Widget& operator=(const Widget& rhs);
	...
}
```

```
Widget w1;                    // default构造函数
Widget w2(w1);                // copy构造函数
w1 = w2;                      // copy assignment操作符

Widget w3 = w2;               // copy构造函数
```

==注意==：w3是新对象被定义，只能是构造，而不能是赋值。

**传值和传引用**：

**函数对象**：重载了operatro()操作符的class。

**未定义/不明确行为**：无法预估运行期发生什么事情。

```
int *p = 0;            // null指针
stdd::cout << *p;      // 对null指针取值，未定义的行为
```



## 1 习惯C++

C++是多重范型编程语言，同时支持过程形式、面向对象形式、函数形式、泛型形式、元编程形式。

**C**：C++以C为基础，很多语法都是来自于C，但是很多时候C++对问题是C的高级解法。

**OO**：class，封装，继承，多态，virtual函数（动态绑定）。

**Template C++**：泛型编程

**STL**：template程序库，容器，迭代器，算法，及函数对象的紧密配合与协调。



## 2 尽量以const，enum，inlin替换#define

```
#define ASPECT_RATIO 1.653
```

其实把#define看作预处理的行为更好行为，而不是语言的一部分。

- ASPECT_RATIO未被编译器所知，没有进入记号表（symbol table），当获得一个编译错误时，给出提示为1.653，很难知晓这个1.653是什么。

而：

```
const double AspectRation = 1.653;
```

作为一个常量，编译器肯定知道它的存在。

- 使用常量比#define占用更小的码，预处理器将ASPECT_RATIO盲目的替换为1.653，导致**目标码**出现多份1.653，而常量AspectRation则不会。

### 以const替换#define的情况

#### 定义常量指针

常量定义式通常放在头文件中，有必要声明为指针常量（指针本身为常量）

```
const char* const authorName = "Scott Meyes";
# 更好是以string对象代替
const std::string authorName("Scott Meyes");
```

#### class专属常量

```
class GamePlayer{
private:
	static const int NumTurns = 5;    // 常量声明式时给定初始值
	int scores[NumTurns];             // 使用该常量
}
```

class的static成员的定义只能在class外部：

```
const int GamePlayer::NumTurns;        // NumTurns定义式，前面给定初值，此时不用给出
```

而#define无法创建class专属常量，#define没有作用域一说，也就没有封装性可言。

### enum

有的编译器不允许在class常量声明时给定初值这种行为如`static const int NumTurns = 5; `，只能在外部定义时给定。而上述`int scores[NumTurns];`语句在编译期间又必须知道`NumTurns`的值，此时使用enum来解决：

```
class GamePlayer{
private:
	enum{NumTurns = 5};               // 令 NumTurns 成为5的一个记号
	int scores[NumTurns];             // 没问题
}
```

但是取一个enum的地址不合法，和#define是一致的。



### inline

很多时候让宏看起来像函数，这种写法不会又额外调用开销，但是会有很多问题：

```
#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a) : (b))
```

这种写法必须为所有实参加上括号，否则调用可能招致奇怪的错误：

```
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);         // a被累加2次, 一次在比较前被累加一次，比较后返回(a)再累加一次，故行为不正常
CALL_WITH_MAX(++a, b+10);      // a被累加1次
```

而使用inline会更好：

```
template<typename T>
inline void callWithMax(const T& a, const T& b){
	f(a > b ? a : b);
}
```



### 总结：

单纯常量，以const或enum代替#define

形似函数的宏，以inline函数带起#define



## 3 尽可能使用const

- 常量指针和指针常量。
- 迭代器和const迭代器（和声明一个T* const指针类似），表明迭代器不能指向其他，而所指内容可以更改。而const_iterator则类似于const T*，表明内容不可更改，但迭代其本身可以更改。
- const和函数返回值配合，可以降低错误：

```
const Rational operator*(const Rational& lhs, const Rational& rhs);
```

而如果不加const：

```
Rational a, b, c;
(a * b) = c;        // 这种用法成为可能
if (a * b = c) ...  // 本意是比较，但是写错，会导致错误，而const返回值就可以避免
```

### const成员函数

1. 对象内容不允许被改变
2. 操作const对象成为可能

两个成员函数，普通成员函数和其对应const成员函数可以算作重载：

```
public TextBlock {
public:
	...
	const char& operator[](std::size_t position) const {   // operator[] for const object
		return text[position];
	}
	char& operator[](std::size_t position) {               // operator[] for non-const object
		return text[position];
	}
private:
	std::string text;
}
```

如下使用：

```
TextBlock tb("hello");
std::cout << tb[0];             // 调用non-const TextBlock::operator[]

const TextBlock c_tb("world");
std::cout << c_tb[0];             // 调用const TextBlock::operator[]
```

所以上述两个重载函数可以根据不同的版本给予不同的返回类型，就可以令const和non-const TextBlock获得不同的处理：

```
std::cout << tb[0];        // ok，读non-const object
tb[0] = 'x';               // ok，写non-const object

std::cout << c_tb[0];      // ok，读const object
c_tb[0] = 'x';             // error，写const object
```

**ERROR**：企图有const版本operator[]返回的const char& 执行赋值动作。

### bitwise constness & logical constness

**bitwise constness**：

成员函数不更改对象的任何成员非static变量，才可以说是const成员函数。可以通过侦测成员变量的赋值动作即可。

但是有的成员函数不满足const性质却可以通过上述bitwise侦测：

```
public TextBlock {
public:
	...
	char& operator[](std::size_t position) const {   // bitwise const声明，但返回非const，其实不适当
		return text[position];
	}
private:
	std::string text;
}
```

**ERROR**：不合适地将operator[]声明为cosnt成员函数，而返回一个reference执行对象内部值，导致成员变量可以被改变的异常行为。

**logical constness**：

上述行为就是logical constness。允许const成员函数可以修改所处理对象的某些bits，但只有在客户端侦测不出的情况下才得如此。

可以通过mutable（可变的）解决。让某些成员变量得以在const成员函数中被改变（坚持bitwise constness），将这些变量声明为mutable。

### const 和 non-const成员函数

为了避免两者函数重复，**让non-const版本调用const版本**，但是得经过一些处理：

```
public TextBlock {
public:
	...
	const char& operator[](std::size_t position) const {   // operator[] for const object
		...
		return text[position];
	}
	char& operator[](std::size_t position) {               // operator[] for non-const object
		return 
			const_cast<char&>(                             // 将op[]返回之的const移除
				static_cast<const TextBlock>(*this)[position] // 为*this对象加上const，得以调用const operator[]
			);                                                // 否则会出现递归调用自己的情况
	}
private:
	std::string text;
}
```

两个转型：

static_cast：首先为*this对象加上const，得以调用const operator[]，否则会出现递归调用自己的情况

const_cast：将op[]返回值的const移除，符合non-const版本



### 总结：

某些声明为const可以由编译器帮助而避免很多错误

编译器强制bitwise constness，但写程序时使用**概念上的常量性**

const和non-const成员函数有等价实现时，non-const调用const避免代码重复



## 4 对象使用前已被初始化

初始化列表：效率更高

一定要使用初始化列表的情况：成员变量const或引用（一定要有初值而不能赋值）



### 总结

为内置对象手工初始化，C++不保证初始化它们

最好使用初始化列表，而不是构造函数中使用赋值操作

避免跨编译单元初始化顺序问题，以local static对象替换non-local static对象

