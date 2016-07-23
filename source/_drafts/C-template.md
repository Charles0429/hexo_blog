---
title: C++ template基础
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

在function template，我们也可以Nontype Template Parameters，表示我们对某个type parameters使用固定类型的参数。

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





