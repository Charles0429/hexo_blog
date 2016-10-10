---
title: C++ template基础
date: 2016-08-07 22:27:27
category: 语言
tags:
  - C++

---

# Introduction

本文总结了C++ templates相关的基础知识，包括如下

- template definition
- template argument deduction

# Template Definition

分为function template和class template。

## Function Template

function template的定义以template关键字开始，后面接着template参数列表，后面接着类似常规的函数定义的语法。举个例子说明

```c++
template <typename T>
int compare(const T &v1, const T &v2)
{
	if (v1 < v2) return -1;
    if (v2 < v1) return 1;
    return 0;
}
```

### Instantiating a Function Template

一般情况下，当我们调用function template时，编译器根据我们提供的参数来自动推导template参数列表，例如

```c++
cout << compare(1, 0) << endl;    // T is int
```

编译器自动推导template参数列表为(T, int)，当template推导出来之后，会自动地去实例化一份函数代码。举上面的int的例子，它会自动地实例化下面的代码

```c++
int compare(const int &v1, const int &v2)
{
	if (v1 < v2) return -1;
    if (v2 < v1) return 1;
    return 0;
}
```

当compare调用其他推导出来的类型时，也会自动地实例化对应的函数版本，例如

```c++
vector<int> vec1{1, 2, 3};
vector<int> vec2{1, 2, 3};
cout << compare(vec1, vec2) << endl;   // T is vector<int>

int compare(const vector<int> &v1, const vector<int> &v2)
{
   //....
}
```

### Template Type Parameters

在function template中，可以使用template type parameters来作为函数参数类型，返回值类型以及函数内部定义类型，例如

```c++
template <typename T> T foo(T* p)
{
	T tmp = *p;
    // ...
    return tmp;
}
```

在较老的C++标准中，还没有typename关键字，之前是用class关键字来当typename用的。不过在支持typename关键字的版本中，还是推荐使用typename。

### Nontype Template Parameters

在function template，我们也可以用Nontype Template Parameters，表示我们对某个type parameters使用固定类型的参数。

在函数实例化时，nontype template parameters应该使用常量表达式作为参数，从而让编译器在编译期间推导出它的值。

举个例子，我们想要比较字符串常量，这些字符串常量以const char开头。因为我们不能拷贝数组，所以，我们的函数参数定义为数组的引用，同时，需要能处理各种不同长度的类型，因此，定义两个nontype template parameters，第一个代表第一个数组的长度，第二个代表第二个数组的长度。

```c++
template <unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M])
{
	return strcmp(p1, p2);
}
```
当我们调用`compare("hi", "mom")`时，会实例化如下代码

```c++
int compare(const char (&p1)[3], const char (&p2)[4]);
```

### inline and constexpr Function Templates

inline和constexpr关键字放在template argument之后，函数返回值之前，如下

```c++
// ok
template <typename T> inline T in(const T &, const T &)

//error
inline template <typename T> T min(const T &, const T &)
```
### Writing Type-Independent Code

从compare可以看出，有两个比较重要的原则可以帮助我们写出更通用的function template

- 函数参数的类型为reference to const
- 比较只用了`<`

使用了reference to const，我们可以保证函数可以用于不能拷贝的类型；只使用`<`，使得我们的函数只要求类型实现了`<`运算符。

为了更进一步地提高通用性，我们可以用`less`函数来进行比较，使得我们的函数对指针也适用

```c++
template <typename T> int compare(const T &v1, const T &v2)
{
	if (less<T>()(v1, v2)) return -1;
    if (less<T>()(v2, v1)) return 1;
    return 0;
}
```

### Template Compilation

模板实例化只在编译器看到了我们使用模板的时候才做，并且实例化的时候，编译器还需要看到模板的代码，因此，一般模板源码放到头文件中。

### Compilation Errors Are Mostly Reported during Instantiation

编译器编译模板代码的三个步骤

1. 编译模板本身，这时候编译器一般可以检查一些语法错误
2. 当编译器看到使用模板时，这个时候会检查一些函数参数个数是否匹配，类型是否一致等信息
3. 当编译器真正实例化时，剩下的编译错误才会被报出来

举个例子

```c++
Sales_data data1, data2;
cout << compare(data1, data2) << endl;
```

这个调用用`Sales_data`来替换T，这里面需要使用`<`，但是`Sales_data`并不支持，因此会报错，但这个错误只有到编译器实例化模板的时候才会报出来。

## Class Template

class template和function template不同的是，class template必须显式地提供模板参数类型。

### Defining a Class Template

先是模板参数列表，然后是class本身，例如

```c++
template <typename T> class Blob {
public:
	typedef T value_type
    typedef typename std::vector<T>::size_type size_type;
    Blob();
    Blob(std::initializer_list<T> i1);
    void push_back(const T &t) {data->push_back(t);}
}
```

### Instantiating a Class Template

为了实例化一个class template，我们需要显式地提供类型信息，例如

```c++
Blob<int> ia; // Blob<int>
Blob<int> ia2 = {0, 1, 2, 3, 4}
```

编译器会实例化以下代码

```c++
template <> class Blob<int> {
	typedef typename std::vector<int>::size_type size_type;
    Blob();
    Blob(std::initializer_list<int> i1);
    int& operator[](size_type i);
    
private:
	std::shared_ptr<std::vector<int>> data;
    void check(size_type i, const std::string &msg) const;
}
```

每个实例化会产生一个独立的class，例如`Blob<string>`跟其他的Blob类型没有任何的关系。

### Member Functions of Class Templates

定义的语法为

```c++
template <typename T>
ret-type Blob<T>::member-name(param-list)
```

一个具体的例子

```c++
template <typename T>
void Blob<T>::check(size_type i, const std::string &msg)
{
	if (i >= data->size()) {
    	throw std::out_of_range(msg);
    }
}
```

### Instantition of Class-Template Member Functions

一般地，只有程序使用了Class Template的成员函数，该成员函数才会被实例化。

### Simplifying Use of a Template Class Name inside Class Code

在一个class template内部，我们可以省略掉模板参数，例如

```c++
template <typename T> class BlobPtr
public:
	BlobPtr(): curr(0) {}
    BlobPtr& operator++()
    BlobPtr& operator--()
```

### Using a Class Template outside the Class Template Body

在class template外部使用时，必须要带上模板参数，例如

```c++
template<typename T>
BlobPtr<T> BlobPtr<T>::operator++(int)
{
	BlobPtr ret = *this
    ++*this
    return ret;
}
```
`BlobPtr ret`就相当于`BlobPtr<T> ret`

### Class Templates and Friends

- 一个class template如果有一个非template类型的友元，那么该友元对于class template的所有实例都生效
- 如果一个class template有template类型的友元，则可以通过控制来决定友元的作用范围

### One-to-One FriendShip

最常见的是友元关系是一个class template和另一个class template以同样模板参数实例化的类互为友元类，例如

```c++
template <typename T> class BlobPtr;
template <typename T> class Blob;
template <typename T>
	bool operator==(const Blob<T>&, const Blob<T> &);
template <typename T> class Blob {
	friend class BlobPtr<T>
    friend bool operator==<T>
    	(const Blob<T> &, const Blob<T> &)
}
```
以相同模板类型初始化的Blob和BlobPtr互为友元类，例如

```c++
Blob<int> ca; // BlobPtr<char> and operator==<char> are friends
BlobPtr<int> ia; // BlobPtr<int> and operator==<int> are friends
```

### General and Specific Template FriendShip

通过控制，还能配置更一般地友元关系，如下

```c++
template <typename T> class Pal;
class C {
	friend class Pal<C>; // Pal<C> is a friend to C
    template <typename T> friend class Pal2; // all instance of Pal2 are friend to C
}

template <tyname T> class C2 {
	friend class Pal<T>;
    template <typename X> friend class Pal2; //all instances of Pal2 are friends of each instance of C2
    friend class Pal3; // Pal3 is friend of every instance of C2
}
```

为了使得所有的实例都是友元，友元的声明必须以不同的模板参数声明。

### Befriending the Template's Own Type Parameter

在C++11标准下，支持以下语法

```c++
template <typename Type> class Bar {
friend Type;
}
```
其中Type可以是内置的类型。

### Template Type Aliases

我们可以使用using语法来创建template别名。

```c++
template <typename T> using twin = pair<T, T>
twin<string> authors;

template <typename T> using partNo = pair<T, unsigned>;
partNo<string> books;
partNo<string> cars;
partNo<Student> kids;
```

### static Members of Class Templates

class template可以定义静态类型，每个实例化的类拥有自身的static成员函数，例如

```c++
template <typename T> class Foo {
public:
	static std::size_t count() { return ctr; }
private:
	static std::size_t ctr;
}
```

## Template Parameters

### Template Parameters and Scope

模板参数使用的名称，在template内部不能再使用。

```c++
typedef double A;
template <typename A, typename B> void f(A a, B b)
{
	A tmp = a; // has the same type with template arugment A
    double = B; // error
}
```

### Template Declarations

template declaration一定要包含template parameters，例如

```c++
template <typename T> int compare(const T&, const T &)
template <typename T> class Blob;
```

### Using Class Members That are Types

假如T是模板参数，那么当编译器看到以下语句时

```c++
T::size_type *p;
```

它需要知道这是定义一个新的指针，还是把size_type和p相乘。默认地，编译器会认为这个不是类型定义，因此，如果是类型定义的话，必须要显式地指明。

```c++
typename T::size_type *p;
```

### Default Template Arguments

可以为模板参数指定默认值，例如

```c++
template<typename T, typename F = less<T>>
int compare(const T &v1, const T &v2, F f = F())
{
	if (f(v1, v2)) return -1;
    if (f(v2, v1)) return 1;
    return 0;
}
```

### Template Default Arguments and Class Templates

当一个class template的所有模板参数都带默认值时，我们定义类时，需要带一个`<>`，例如

```c++
template <class T = int> class Numbers {
public:
	Numbers(T v = 0) : val(v) {}
private:
	T val;
}
Numbers<long double> lots_precision;
Numbers<> average_precision; // empty, T = int
```

## Member Templates

成员函数本身也可能是模板，分为class template和non class template两种情况讨论。

### Member Templates of Ordinary(Nontemplate) Classes

举个例子，和unique_ptr的默认删除器的实现有关，如下

```c++
class DebugDelete {
public:
	DebugDelete(std::ostream &s = std::cerr) : os(s) {}
    template <typename T> void operator()(T *p) const 
    {
    	os << "deleting unique_str" << std::endl;
        delete p;
    }
private:
	std::ostream &os;
}
```

使用方法如下：

```c++
double *p = new double;
DebugDelete d;
d(p);     // calls DebugDelete::operator()(double *)
int *ip = new int;
DebugDelete()(ip); //operator()(int *)
```

### Member Templates of Class Templates

和class template的普通函数不同，member template是function template，在定义时，还得带上函数本身的模板参数，如下

```c++
template <typename T> class Blob {
	template <typename It> Blob(It b, It e);
}

template <typename T>
template <typename It>
Blob<T>::Blob(It b, It e):
	data(std::make_shared<std::vector<T>(b, e)>) {
        
}
```
 
### Instantiation and Member Templates

对于member template，对于类的模板参数是要指定的，而其本身的模板参数一般是通过函数参数推断出来的。

## Controlling Instantiations

当两个或多个独立的源代码文件使用了相同参数的模板，每个源代码文件中都会有该模板的一份实例化的代码。

在大型项目中，这会造成代码体积变大，编译变慢。在C++11中，可以通过如下方法来避免该问题

```c++
extern template declaration;

template declaration;
```
一般地，可以把模板实例化的代码放到一个单独的文件中，如下

```c++
template int compare(const int&, const int&);
template class Blob<string>;
```

需要注意的是，这种声明会把整个类中的所有函数都实例化。

# Template Argument Deduction
主要描述function template的参数推导规则。

## Conversions and Template Type Parameters

编译器实例化模板时，会考虑以下转换规则：

- const转换：当函数参数是const引用时，一个非const对象是可以作为参数传入的
- 数组或函数到指针的转换：一个数组会被转换成一个元素的指针，函数则会转成函数指针

例如

```c++
template <typename T> T fobj(T, T);
template <typename T> T fref(const T&, const T&);
string s1("a value")
const string s2("another value")

fobj(s1, s2);  // fobj(string, string)，const被忽略
fref(s1, s2); // fref(const string &, const string &)

int a[10], b[42];
fobj(a, b); // f(int *, int *)
fref(a, b); // error：数组无法转换成引用
```

第一个fobj(s1,s2)，const能忽略的原因是，因为s1和s2都会拷贝到函数参数中，所以，原来是否是const不影响。

### Function Parameters That Use the Same Template Parameter Type

以前面的compare函数为例

```c++
long lng;
compare(lng, 1024); //错误：无法实例化compare(long, int)
```

### Normal Conversions Apply for Ordinary Arguments

普通的参数没有特殊的转换，跟之前的函数转换规则一样。

## Function Template Explicit Arguments

function template也可以显式地指定模板参数。

### Specifying an Explicit Template Argument

例如，一个加法运算，用户可以指定返回值类型，来控制运算的精度。

```c++
template <typename T1, typename T2, typename T3> T1 sum(T2, T3);
```
在这种情况下，T1是无法被推导出来的，只能是调用者显式地指定。

### Normal Conversions Apply for Explicitly Sepecified Arguments

对于指定了类型的模板函数，是可以采用通用的类型转换规则的

```c++
long lng;
compare(lng, 1024);  // error
compare<long>(lng, 1024); // ok
compare<int>(lng, 1024); //ok
```

## Trailing Return Type and Type Transformation

可以使用尾置返回类型，从函数参数中推导出返回类型，从而避免用户需要显式地指定返回类型，例如

```c++
template <typename It>
auto fcn(It beg, It end) -> decltype(*beg)
{
	//process
    return *beg;
}
```

上述函数中，返回类型为beg指向的类型的引用。

### The Type Transformation Library Template Classes

可以通过`remove_reference`来实现函数返回一个值，而非引用，如下

```c++
template <typename It>
auto fcn2(It beg, It end) ->
	typename remove_reference<decltype(*beg)>::type
{
	//process the range
    return *beg;
}
```

除了`remove_reference`，我们还可以有`remove_pointer`等实现各种功能的type transformation函数。

## Function pointers and Argument Deduction

通过函数指针赋值，可以直接实例化一个模板函数，如下

```c++
template <typename T> int compare(const T&, const T&);
int (*pf1)(const int&, const int&) = compare
```
上面会直接实例化参数T为int的compare函数

## Template Argument Deduction and References

分函数参数为左值和右值两种情况讨论。

### Type Deduction from Lvalue Reference Function Parameters

当函数参数为左值引用时，可以传入的参数为左值，可以为const类型，如下

```c++
template <typename T> void f1(T &);
f1(i); // T is int
f1(ci); // T is const int
f1(5); // error must be lvalue
```

### Type Deduction from Rvalue Reference Function Parameters

当函数参数为右值引用时，如下

```c++
template <typename T> void f3(T&&);
f3(42); // T is int
```

### Reference Collapsing and Rvalue Reference Parameters

当传入一个int左值到f3函数时，正常情况下来看，由于i是左值，应该不能绑定到右值参数上。但是，有两个特殊的绑定规则，可以支持传入左值：

- 当我们传入左值到右值引用时，模板参数类型会被推导成左值引用
- 由于传入的参数被推导成引用，而函数参数也是引用，会产生引用堆叠，这个有特殊的规则

引用堆叠的规则

- X& &，X& &&和X&& &会被看成X&
- X&& &&会被看成 X&&

### Writing Template Functions with Rvalue Reference Parameters

上述规则会造成以下代码产生奇怪的现象

```c++
template <typename T> void f3(T&& val)
{
	T t = val;
    t = fcnt(t);
    if (val == t) {}
}
```

- 当传入的参数是右值时，T是int
- 当传入的参数是左值时，T是int&

一般右值引用参数用于两种场景，参数转发和模板重载。

## Understanding `std::move`

std::move的定义如下

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
	return static_cast<typename remove_feference<T>::type&&>(t);
}
```

std::move的工作原理：

当传入右值时，例如string("test")，其工作流程如下

- 推导出来的类型T为string
- 因此，remove_reference实例化为string
- remove_reference<string>为string
- move的返回类型为string &&
- 而move的函数参数t，为string &&

这时候实例化的move函数为

```c++
string && move(string &&t);
```
函数返回`return static_cast<string &&>(t)`，由于t本身就是`string &&`，因此，函数会返回传入的右值引用。

当传入左值时，其工作流程如下

- 推导出来的类型T为string &
- remove_reference实例化为string &
- remove_reference<string &>为string
- move的返回类型还是string &&
- move的函数参数，实例化为string & &&，堆叠成string &

这次调用会实例化move函数为

```c++
string&& move(string &t);
```
函数返回值为`static_cast<string &&>(t)`，因为t的类型为`string &`，因此，会转换成`string &&`。

## Forwarding

有些函数需要转发一些参数，且要保留它们的类型信息，包括左值右值，const类型信息等。

以一个例子来说明，如下

```c++
template <typename F, typename T1, typename T2>
void flip1(F f, T1 t1, T2 t2)
{
	f(t2, t1);
}
```
当f中有引用类型时，会有奇怪的现象

```c++
void f(int v1, int &v2)
{
	cout << v1 << " " << ++v2 << endl;
}

f(42, i); //f改变参数i
flip1(f, j, 42); //flip1并没有改变j
```
flip1推导出T1是int，然后，传递给f的是函数左值参数，因此，并不会改变外面传进入的j。

即，如下

```c++
void flip1(void (*fcn)(int, int &), int t1, int t2);
```

### Defining Function Parameters That Retain Type Information

可以使用右值引用来解决上述问题，如下

```c++
template <typename F, typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2)
{
	f(t2, t1);
}
```
当再次考虑调用如下是

```c++
flip1(f, j, 42);
```

其中T1会被推导成`int &`，然后根据堆叠规则，t1的类型为`int &`，因此，在f中是能增加j的值的。

上述函数在左值时能正常工作，但是，在右值引用时，**无法正常工作**，如下

```c++
void g(int &&i, int &j)
{
	cout << i << " " << j << endl;
}

flip2(g, i, 42); //error
```
其中，42推导出T2为int，然后t2是 `int &&`，注意，只是类型是`int &&`，但它本身是一个lvalue，所以，不能绑定到g的第一个参数。

### Using `std::forward` to Preserve Type Information in a Call

可以使用std::forward解决上述问题，`std::forward<T>`的返回值为`T &&`，如下

```c++
template <typename Type> intermediary(Type &&arg)
{
	finalFcn(std::forward<Type>(arg));
}
```

- 当传入的参数是是rvalue，Type的类型是rvalue的类型，那么，`forward<Type>`将返回`Type &&`
- 当出软的参数是lvalue时，那么Type是lvalue reference，即Type &，则`forward<Type>`则是`&&&`堆叠，最后，返回的还是lvalue reference

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://o8m1nd933.bkt.clouddn.com/blog/qcode_wechat.jpg)

# References

- C++ primer 5th edition





