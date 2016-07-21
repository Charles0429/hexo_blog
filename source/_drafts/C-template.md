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




