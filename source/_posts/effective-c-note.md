---
title: Effective C++ 学习笔记
date: 2016-06-05 15:38:33
categories: 语言
tags:
  - C/C++
  - 学习笔记
---

# Effective C++学习笔记

## 1. 导读（基础部分）

### 1.1 视C++为一个语言联邦

C++语言的四个层次：

- C。没有C++的面向对象，没有模板，没有异常，没有重载等。
- Object-Oriented C++。这部分也就是C with Classes。classes、封装、继承、多态、虚函数。这部分是面向对象的特性。
- Template C++。这部分是C++的泛型编程部分。这部分带来的是template metaprogramming，也就是所谓的模板元编程。
- STL。STL是个template程序库。它对容器、迭代器、算法及函数对象的规约，并且是以templates及程序库的方式构建出来。

每个层次应该有自己的最佳实践。例如对于C层次，传入函数最佳的实践应该是传入值，而不是指针，而对于C with classes层次，则以传递引用为最佳的实践。

### 1.2 尽量以const、enum、inline替换define

- 对于全局的，用define定义的值，在预处理的时候会被替换成相应的值，这对于调试的符号表不利。并且用define的符号在多处使用都会要替换，会使编译生成的代码量变大。


- 对于class的常量，为了只提供一份定义，一般用static const。用define无法创建一个class专属常量，因为#define并不重视作用于，一旦宏被定义，它就在其后的编译过程中有效。有时候，为了不让获取常量的定义地址，则可以用enum来代替#define

  ```
  class Test
  {
  enum {NumTurns = 5};
  int scores[NumTurns];
  };
  ```

- 有时候需要用模板函数来代替宏

  宏的一些副作用：

  ```
  #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
  int a = 5, b = 0;
  CALL_WITH_MAX(++a, b); //a被累加两次
  CALL_WITH_MAX(++a, b+10); //a被累加一次
  ```

  这时候可以用模板函数来代替

  ```
  template<typename T>
  inline void callWithMax(const T &a, const T &b)
  {
    f(a > b ? a : b);
  }
  ```

  有了consts、enums和inlines，我们对预处理器的需求降低了，但是不等于没有。#include仍然是必需品，而#ifdef/#ifndef也继续扮演控制编译的重要角色。

### 1.3 尽可能使用const

- 令函数返回一个常量值，往往可以降低因客户端而造成的意外

  ```
  const Rational operator*(const Rational &lhs, const Rational &rhs);
  ```

  这样可以避免

  ```
  if ((a * b) = c)这样的错误。
  ```

- const成员函数

  对于成员函数的const，有两个概念：bitwise constness和logical constness。

  bitwise constness：指的是成员函数只有在不更改对象之任何成员变量才可以说是const。也就是说它不更改对象内的任何一个bit。不幸的是许多成员函数虽然不十足具备const性质却能通过bitwise测试。例如，一个更改了“指针所指物”的成员函数虽然不能算法const，但如果只有指针率属于对象，那么此函数不会引发编译器意义，这导致反直观结果。

  ```
  class CTextBlock
  {
  public:
  char &opeartor[](std::size_t position) const //bitwise const声明，但其实不适当
  private:
  char *pText;
  };
  cosnt CTextBlock cctb("Hello");
  char *pc = &cctb[0];
  *pc = 'J';
  ```

  你创建一个常量对象设以某值，而且只对它调用const成员函数。但你还是改变了它的值。

  ​这种情况导出所谓的logical constness。这一派拥护者主张，一个const成员函数可以修改它所处理的对象内的某些bits，但只有在客户端侦测不出的情况下才得如此。

```
class CTextBlock
{
public:
	std::size_t length() const;
private:
	char *pText;
    std::size_t textLength;
    bool lengthIsValid;
};
std::size_t CTextBlock::length() const
{
  if (!lengthIsValid)
  {
  	textLength = std::strlen(pText);
    lengthIsValid = true;
  }
  return textLength;
}
```

length的实现当然不是bitwise const，因为textLength和lengthIsValid都可能被修改。这两笔数据被修改对于const CTextBlock对象虽然可接受，但编译器不同意。它们坚持bitwise constness。

解决方案很接单：把它们设置成mutable。

- const和non const防止重复

  ```
  const char &operator[](std::size_t position) const
  {
    ...   //边界检验
    ...   //日志记录访问记录
    ...   //检验数据完整性
    return text[position];
  }

  char &operator[](std::size_t position)
  {
    ...  //边界检验
    ...  //日志记录访问记录
    ...  //检验数据完整性
    return text[postion];
  }
  ```

  这里用non const函数调用const函数的方法来避免代码重复，这种方式并不推荐。

  用const调用non const的方法是不合适的，因为non const方法有可能修改数据，而const调用之后，就可能也会修改数据，不符合函数的const语义。

### 1.4 确定对象被使用前已被初始化

- 对于内置类型，定义的时候一定要初始化，因为读取为初始化的值是未定义的
- 对于非内置类型，初始化责任落在构造函数上。确保每一个构造函数都讲对象的每一个成员初始化。

**注意不要混淆赋值和初始化**

```
ABEntry::ABEntry(const std::string &name, const std::string &address,
	const std::list<PhoneNumber> &phones)
{
	theName = name;       //这些都是赋值
    theAddress = address; //而非初始化
    thePhones = phones;
    numTimesConsulted = 0;
}
```

这里，进行赋值之前，会调用成员的默认构造函数，然后才是赋值。这样性能不够好。可以换成如下：

```
ABEntry::ABEntry(const std::string &name, const std::string &address,
	const std::list<PhoneNumber> &phones)
  : theName(name),
  	theAddress(address),
    thePhones(phones),
    numTimesConsulted(0)
{
}
```

另外初始化列表中，成员话初始化的次序是类中定义的变量次序一样的。

- 对于不同编译单元的non local static对象的初始化顺序是未定义的

  所谓编译单元是指产出单一目标文件的那些源码。基本上它是单一源码文件加上其所含入得头文件。

  static对象，其寿命从被构造出来知道程序结束为止。

  non-local，指的是对象位于global或位于namespace作用于，抑或是class内或file作用于内被声明为static

  ```
  class FileSystem
  {
  public:
  std::size_t numDisks() const;
  }
  extern FileSystem tfs;
  ```

  另一个文件：

  ```
  class Director
  {
  public:
  Directory(params);
      ...
  }
  Director::Director(params)
  {
    std::size_t disks = tfs.numDisks(); //使用tfs对象
  }
  ```

  现在客户端决定创建一个Directory对象，

  ```
  Directory tempDir(params);
  ```

  现在，初始化次序的重要性显现出来了：除非tfs在tempDir之前先被初始化，否则会用到尚未初始化的tfs。

  ​

  ​解决方案：

  ​把每个non-local static对象搬到自己的专属函数内，这些函数返回一个refrence指向它所含的对象。这个是单例模式一个常见的实现手法。

```
FileSystem &tfs()
{
  static FileSystem fs;
  return fs;
}
Director &tempDir()
{
  static Director td;
  return td;
}
```

使用static对象可能会在多线程环境下造成race condition。



## 2. 构造/析构/赋值运算

### 2.1 了解C++默默编写并调用哪些函数

- 如果没有自己编写构造函数，C++会生成一个不带参数的默认构造函数。

- 在非特殊情况下，C++会自动生成拷贝构造函数、赋值运算符以及析构函数

  特殊情况指的是：如果类中有引用类型或者有const类型，此时由于引用类型和const类型不能重新赋值，所以编译器这个时候不会自动生成赋值运算符和拷贝构造函数。

### 2.2 若不想使用编译器自动生成的函数，就该明确拒绝

如果一个类不希望被拷贝，那么其拷贝构造函数和赋值函数应该不生效。对于外部调用来讲，将拷贝构造函数和赋值函数声明为private就可以了。但是，如果类的成员函数和友元来调用，还是可以通过编译的，这个时候，可以只声明private类型的拷贝构造函数和赋值函数，而不去实现它们，这样友元或成员函数试图调用它们的时候，就会报链接错误。

如果希望在编译的时候就讲问题暴露出来，可以使用如下方法：

```
class Uncopyable
{
protected:
	Uncopyable() {}
    ~Uncopyable() {}
private:
	Uncopyable(const Uncopyable &);
    Uncopyable &operator=(const Uncopyable);
};
```

为求阻止HomeForSale对象被拷贝，我们唯一需要做的就是集成Uncopyable。

当成员函数或者友元函数，尝试拷贝HomeForSale对象，编译器变试着生成一个copy构造函数和一个copy assignment操作符，这样会去调用其base class的对应函数，这会被编译器拒绝，因为base class的拷贝函数是private。

### 2.3 为多态基类声明virtual析构函数

对于一个多态类的基类，其析构函数应该声明为virtual函数

例如

```
class Base
{
private:
	int a;
};
class Derived
{
private:
	int b;
};
Base *p = new Derived();
delete p;
```

如上所示代码，由于p是一个基类的指针，而且p的析构函数是普通的函数，所以，其析构的时候，只会按照Base类的方式析构，而Dervied类部分的数据往往会没有清除掉。这会造成局部的资源泄漏、败坏数据结构等。

正确的方法应该是为一个多态基类声明一个non virtual的析构函数，这样基类指针指向子类的时候，会自动调用子类的析构函数，这样就能完全的析构子类的资源。

当然，如果一个基类不是为了实现多态，那么就没有必要将析构函数声明为虚函数，这样会有空间上的浪费。因为有虚函数的类，会有一个指向函数指针数组的指针，会占用sizeof(void *)的空间。

例如，对于std::string，那么如果一个类要继承它，就绝对不能用std::string类型的指针来指向子类，否则就可能造成上述的资源泄漏。



有时候，class带pure virtual函数，可以方便的实现纯虚类，这时候，往往可以来定义一个pure virtual的析构函数，不过你得对纯虚的析构函数做一份定义，因为它的子类析构的时候回调用到它。

### 2.4 别让异常逃离析构函数

假设有一个vector<Test>数组，其正在析构，假如其第一个元素析构过程中抛出异常，即使这样，其他元素的空间还是要释放的，所以，继续析构第二个元素，假如其中又抛出了异常，这时候，对于C++来讲，抛出两个异常时未定义的行为，这时候只能结束程序，所以，在C++中，别让异常逃离析构函数。

那么，应该怎么样实现不让异常逃离析构函数呢？

假设一个数据库连接的类，为了防止客户忘记关闭连接，往往会在析构的时候尝试关闭连接，例如

```
class DBCONN
{
~DBCONN()
{
  try
  {
  	db.close();
  }catch 
  {
  	//记下错误，继续执行或者调用abort，终止程序
  }
}
private:
	DB db;
};
```

这种处理方式的缺点是，客户没有太多选择，要么是abort终止程序，要么是继续执行。

为了让客户有选择，可以单独提供一个close函数，让用户自己处理异常。如果用户决定自己不处理，那么析构函数的时候再采用一个默认的方式来处理。

### 2.5 不要在构造函数和析构函数中调用虚函数

在构造函数中，调用构造函数的顺序是基类->子类，当基类在构造的时候，子类的部分还没有开始构造，这时候，如果调用虚函数，只会调用基类版本的，不符合虚函数的语义。

在析构函数中，调用析构函数的顺序是子类->基类，当基类在析构的时候，子类的部分已经析构完成，这时候，如果调用虚函数，同样只会调用基类版本的，不符合虚函数的语义。

### 2.6 令operator=返回一直refrence to *this

一般赋值操作，希望可以写成连续的形式

```
int x, y, z;
x = y = z = 15;
```

对于上述形式，则必须返回一个refrence to *this。

类似的，*=， +=等等都应该返回refrence to *this。

### 2.7 在operator=中处理“自我赋值”

对于防止自我赋值很有必要

```
Widget w;
w = w;
a[i] = a[j]; //a[i]和a[j]实际上指向同一个元素
*pi = *pj; //pi和pj实际上指向同一个元素
```

自我赋值的危害

```
Widget
{
private:
	Test *p;
};
Widget &Widget::operator=(const Widget &w)
{
  delete p;
  p = new int (*w.p);
  return *this;
}
```

如果是自我赋值，上面的代码会把自己的空间释放掉，然后又再次引用，出现未定义的行为。

改良版本

```
Widget &Widget::operator=(const Widget &w)
{
  if (this == &w)
  {
  	return *this;
  }
  delete p;
  p = new Test(*w.p);
  return *this;
};
```

这个类虽然是能避免自我赋值问题，但是，如果new Test抛出了异常，那么Widget最终会持有一个指针指向一块被删除的Test，这样的指针是有害的。

```
Widget &Widget::operator=(const Widget &rhs)
{
  Test *porg = p;
  p = new Test(*rhs.p);
  delete porg;
  return *this;
}
```

如果此时new Test发生异常，那么p可以保持原状。（不明白异常发生的时候，不会给p赋值吗？）

还有一个可以避免异常的方法

```
Widget &Widget::operator=(const Widget &rhs)
{
  Widget tmp(rhs);
  swap(tmp);
  return *this;
}
```

上面的代码确保异常安全，并且能够解决自我赋值问题。



### 2.8 复制对象时勿忘其每一个成分

考虑两个问题：

- 有一个类，添加了一个成员变量，但是有可能忘记在拷贝构造函数还有赋值函数中把这个成员变量添加进去，这回造成部分拷贝，而且编译器还不会有任何的提示。
- 有一个基类和子类，子类的拷贝构造函数和赋值函数，可能会忘记给基类调用相应的拷贝构造函数，和赋值函数，这也会造成部分资源没有正确拷贝。

所以，复制对象时勿忘其每一个成分，添加了成员变量要记得修改相应的拷贝构造函数和赋值函数。对于子类，在其拷贝构造函数和赋值函数中，记得要调用相应的基类版本。



**不要尝试以某个copying函数实现另一个copying函数，应该讲共同机能放入第三个函数中，并且由两个copying函数共同调用。** 备注：还不太理解这样做的目的。

## 3 资源管理

### 3.1 以对象管理资源

有以下代码：

```
void function()
{
  Widget *w = new Widget();
  if (xxx)
  {
    return;
  }
  delete w;
}
```

如上代码所示，在function中创建了一个widget对象，在函数结束时，应该把创建对象释放掉。但往往有时候新添加代码的时候，可能忘记在return语句前加入释放空间的语句了。

一般这种可以用RAII来解决，RAII思想指的是

- 获得资源后立刻放进资源管理对象内
- 管理对象运用析构函数确保资源被释放

```c++
class Resource{};
class RAII{
public:
    RAII(Resource* aResource):r_(aResource){} //获取资源
    ~RAII() {delete r_;} //释放资源
    Resource* get()    {return r_ ;} //访问资源
private:
    Resource* r_;
};
```

### 3.2 在资源管理类中小心copying行为

对于资源管理类的copying行为，可能需要按照以下方面来决定：

- 禁止复制。有的RAII对象复制并不合理，如资源管理类中管理的锁
- 对底层资源祭出“引用计数法”。有时候，我们希望保持资源，直到它的最后一个使用者被销毁，例如shared_ptr。
- 复制底层资源。有时候，资源管理需要对某一份资源的任意数量的副本，而你需要资源管理类的唯一理由是，当你不在需要某个副本的时候，就释放它的空间。
- 转移底部资源的拥有权。确保RAII对象指向一个未加工的资源。即使复制，此时资源的拥有权从被复制物转移到目标物。（备注：这种情况应该比较少见吧）。

### 3.3 在资源管理类中提供对原始资源的访问

很多API可能会用到原始的资源形式，例如pthread库中，condition的wait函数就需要mutex *，所以，需要RAII中需要提供一个接口对原始资源。

例如，对于mutex，我们可以用get_mutex来返回对原始mutex的引用。

有时候可能也会提供隐式转换的接口，例如：

```
operator mutex() const;
```

这样就会在需要mutex的时候，自动转换。

### 3.4 成对使用new和delete时采取相同形式

首先，来看new和delete操作符的语义

- new会先创建对象的内存空间，然后调用其构造函数来初始化
- delete会先调用析构函数，然后再释放类的空间

然后，new和delete需要采用相同的形式

```
std::string *str_ptr1 = new std::string;
std::string *str_ptr2 = new std::string[100];
...
delete str_ptr1;
delete[] str_ptr2;
```

如果用delete[]来删除str_ptr1，结果是未定义的，delete可能会先解析出数组大小，然后，析构这么多个元素，但是实际上只有一个元素需要释放。同样地，对str_ptr2调用delete，也会出现未定义的行为。

因此，new和delete形式必须要以相同的形式出现。

```
new和delete
new[]和delete[]
```

### 3.5 以独立语句将newed对象置入智能指针

考虑以下代码

```
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```

由于C++中，一条语句里面的函数调用的顺序是不确定的，上面的代码总共有三条语句：

```
new Widget
priority()
std::tr1::shared_ptr<Widget>();
```

如果最终编译器以上述顺序执行，并且priority函数中间发生了异常，那么new Widget创建的对象还没加入到shared_ptr中，会造成资源泄漏。

正确的做法

```
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

## 4 设计与声明

### 4.1 让接口容易被正确使用，不易被误用

- 要设计不易被误用的接口

例如，要创建一个时间类型

```
Date(int month, int year, int day);
```

上面的接口就是容易被误用的接口，因为可能把年月日的顺序给搞错了。

可以给年月日分别设计一个类型，构造函数中传入对应的类型才能正确编译。

- 限制类型内什么事可做，什么事不能做

  比如opeator *的返回值类型为const

- 让你自己设计类型的表现与内置types一致

- 任何借口如果要求客户必须记得做某事，就容易出错，最好的方法就是给客户做好

  例如

  ```
  Investment *createInvestment();
  ```

  该函数返回一个指针，需要客户在最后记得释放空间，当客户忘记这件事时，就会造成资源未释放。

  比较好的方法，是返回一个shared_ptr智能指针。

- shared_ptr支持定制型删除器，可以返回跨模块的new/delete问题，因为shared_ptr会自动调用本模块的delete函数。

### 4.2 设计class犹如设计type

- 新type的对象应该如何被创建和销毁

- 对象的初始化和对象的赋值该有什么样的差别

- 新type的对象如果被passed by value，意味着什么？即copy构造函数该如何实现

- 什么是新type的“合法值”？成员函数必须对菲合法值进行必要的错误检查

- 你的新type需要配合某个继承图系？

- 你的新type需要什么样的转换？是否需要隐式或者显式转换成其他类型。

- 什么样的操作符和函数对此新type而言是合理的?该声明哪些函数，哪些应该是member函数，某些则不是。

- 什么样的标准含税应该驳回？例如，自动生成的拷贝构造函数，赋值函数和析构函数等等。

- 谁该取用新type的成员？哪些为public、哪些为protected、哪些为private。

- 什么是新type的“未声明接口”？这个不太懂

- 你的新type有多么一般化？或许你其实并非定义一个新type，而是定义一整个

  types家族。

- 你真的需要一个新type吗？如果只是定义新的derived class以便为既有的class添加机能，那么说不定单纯定义一个或多个non-member函数或者template更能达到目标。

### 4.3 宁以pass-by-refrence-to-const替换pass-by-value

对于自定义的类，往往通过pass by const refrence比较合适，因为这可以省去多次的构造函数的调用开销。对于内置对象，往往以pass-by-value比较合适，因为引用在编译器内部实现往往是指针，对于内置类型，用指针来传递往往会比pass-by-value变慢，多了一次读内存的过程（先读变量的地址，然后在真正读变量的值）。

对于STL迭代器和函数对象，往往习惯上也被设计为pass-by-value。

### 4.4 必须返回对象时，别妄想返回其refrence

举个例子

```c++
const Rational &operator(const Rational &lhs, const Rational &rhs);
```

这个要返回refrence，所以，必须在heap上分配一个对象。因为，refrence指向一个函数内的local对象时，在函数退出时，已经被析构了，这时候，函数的返回值就是非法的了。

当你在必须“返回一个refrence和返回一个object”之间进行抉择时，你的工作就是挑出行为正确的那个。



绝不要返回pointer或refrence指向一个local stack对象，或返回refrence指向一个heap-allocated对象（这个需要外部调用delete，而且如果连续赋值可能导致内存空间无法释放），或返回pointer或refrence指向一个local static对象而有可能同时需要对个这样的对象。

### 4.5 将成员变量声明为private

- 如果将成员变量声明为public，那么，如果对成员变量进行改动，会影响到所有使用该成员变量的客户。
- 如果将成员变量声明为protected，那么，如果对成员变量进行改动，会影响到所有使用该成员变量的derived classes。

所以，为了封装性，把成员变量声明为private。

### 4.6 宁以non-member-non-friend替换member函数

```c++
class WebBrowser
{
public:
	void clearCache();
    void clearHistory();
    void removeCookies();
};
class WebBrowser
{
public:
	void clearEverything();
};
void clearBrowser(WebBrowser &wb)
{
  wb.clearCache();
  wb.clearHistory();
  wb.removeCookies();
};
```

从上面代码看出，以non-member-non-friend函数方式形式来实现clear操作会更有封装性，因为它不能访问类里面的数据，更加符合封装的思想。

### 4.7 若所有参数皆需类型转换，请为此采用non-member函数

```
class Rational
{
public:
	Rational(int numerator = 0, int denominator = 1);
    int numberator() const;
    int denominator() const;
private:
	...
};
class Rational
{
public:
	const Rational operator* (const Rational &rhs) const;
};
```

对于上面代码

```
Rational result = oneHalf * oneEighth;
result = result * oneEighth;
result = oneHalf * 2;
result = 2 * oneHalf; //有错误
```

为了使得第四个赋值也能支持，可以把operator*实现成以下

```
const Rational operator*(const Rational &lhs, const Rational &rhs);
```

### 4.7 考虑写出一个不抛异常的swap函数

这部分没有怎么看懂。

## 5 实现

### 5.1 尽可能延后变量定义式的时间

变量的定义最好要出现在其初值能确定的地方。这样可以避免定义了变量，未使用，却带来了构造函数和析构函数的开销。

对于循环：

```
//方法A：定义循环外
Widget w;
for (int i = 0; i < n; ++i)
{
  w = 取决于i的某个值;
}
//方法B：定义循环内
for (int i = 0; i < n; ++i)
{
  Widget w(取决于i的某个值);
}
```

- 做法A：1个构造函数+1个析构函数+n个赋值函数
- 做法B：n个构造函数+n个析构函数

如果classes的一个赋值成本低于一组构造+析构成本，做法A大体而言比较高效。尤其当n比较大的时候。否则做法B或许比较好。

另外A带的是w的作用域变大，有时候会对程序的可理解性和易维护性造成冲突。

因此：

- 你知道赋值成本比“构造+析构”成本低
- 你正在处理代码中效率高度敏感的部分，否则你应该使用做法B

### 5.2 避免使用转型(cast)

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免，如果有个设计需要转型动作，试着发展无需转型的替代设计。
- 如果转型是必要的，试着将它隐藏于某个函数背后，客户随后可以调用该函数，而不需将转型放进他们自己的代码内。
- 宁可使用C++风格转型，不要使用旧式转型前者很容易辨识出来，而且也比较有分别别类的职称。

### 5.3 便面返回handles指向对象内部

- 帮助const成员函数行为像个const

```c++
class Rectangle
{
public:
	const Point &upperLeft() const {return pData->ulhc;}
    const Point &lowerRight() const {return pData->lrhc;}
};
```

如果这里用非const返回值，会造成Rectangle中的ulhc货lrhc被外部的修改。

- 避免返回handles(包括refrence、指针、迭代器)指向对象内部。遵守这个条款可增加封装性。

因为有可能在某个临时对象上，操作这个内部对象，会造成问题：

例如：

```
GUIObject *pgo;
const Point *pUpperLeft = &(boundingBox(*pgo).upperLeft());
```

这个boundingBox会生成一个临时的Rectangle对象，造成upperLeft的返回的内部对象失效，造成程序不安全。

### 5.4 为“异常安全”而努力是值得的

异常安全性，后面再看。

### 5.5 透彻了解inlining的内内外外

- inline函数如果起作用了，会在每次调用的时候用实现的代码替换它，所以会造成程序体积变大。
- inline只是向编译器的一个申请，可以明确用inline关键字，也可以在类体内部定义。
- template的实例化和inline无关，如果需要template函数为inline，需要显式地声明它。
- inline函数无法随着程序库的升级而升级。换句话说如果f是程序库内的一个inline函数，客户讲“f函数本体”编进其程序中，一旦程序库设计者决定改变f，所有用到f的客户端程序都必须重新编译。
- 将大多数inline限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。

### 5.6 将文件间的编译依存关系降到最低

- 支持“编译依存性最小化”的一般构想是：相依与声明式，不要相依与定义式，基于此构想的两个手段是Handle classes和Interface Classes。

## 6 类的继承

### 6.1 确定你的public继承塑模出is-a关系

适用于base classes身上的每一件事情一定也适用于derived classes身上，因为每一个derived class对象也都是一个base class 对象。

### 6.2 避免遮掩继承而来的名称

看以下代码：

```c++
class Base
{
private:
	int x;
public:
	virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
    ...
};
class Derived : public Base
{
public:
	virtual void mf1();
    void mf3();
    void mf4();
}；
Derived d;
int x;
...
d.mf1(); //没问题，调用Dervied::mf1
d.mf1(x); //错误，因为Derived::mf1遮掩了Base::mf1
d.mf2(); //没问题，调用Base::mf2
d.mf3(); //没问题，调用Dervied::mf3
d.mf3(x); //错误！因为Derived::mf3遮掩了Base::mf3
```

- 如果你正在使用public继承而又不继承那些重载函数，就是违反Base和derived classes之间的is-a关系，而is-a是public的基石。


- 为了被遮掩的名称再见天日，可使用using声明式

### 6.3 区分接口继承和实现继承

考虑如下一段代码

```
class Shape
{
public:
	virtual void draw() const = 0;
    virtual void error(const std::string &msg);
    int objectID() const;
    ...
};
class Rectangle : public Shape {...};
class Ellipse : public Shape {...};
```

- 对于一个pure virtual函数，且不带定义，目的是为了让derived classes只继承函数接口。然后，继承类必须实现它自身的行为。
- 声明impure virtual函数的目的是，让derived classes继承该函数的接口和缺省实现。也就是，如果某个集成类不想对该功能做特殊处理的时候，可以采用缺省的默认实现，否则，需要自己提供实现。这种方法可能会造成继承类忘记实现自己该实现的功能，这时候编译照样是过的。
- 声明pure virtual函数且带定义，也是让derived classes继承该函数的接口和缺省实现。这种方法跟上面的区别是，它可以防止集成类忘记实现自己功能的情况，因为是pure virual，采用默认实现也得显式地调用基类的函数。
- 声明non-virtual函数的目的是为了令derived classes继承函数的接口及一份强制性实现。

### 6.4 考虑virtual函数以外的其他选择

场景：假如你打算为游戏内的人物设计一个继承体系。你的游戏属于暴力砍杀类型，剧中人物被伤害或因其他因素而降低健康状态的情况并不罕见。因此，你决定提供一个成员函数healthValue，它会返回一个整数，表示人物的健康程度。由于不同的人物可能已不同的方式计算它们的健康指数，将healthValue声明为virtual似乎是再明白不过的做法。

#### 方案1-虚函数方法

```c++
class GameCharacter
{
public:
	virtual int healthValue() const; //返回人物的健康指数
                                     //derived classes可重新定义它
};
```

#### 方案2-借由Non-Virtual Interface手法实现Template Method模式

这个方案的思路是保留healthValue为public成员函数，但让它成为non-virtual，表调用一个private virtual函数，进行实际工作：

```c++
class GameCharacter
{
public:
	int healthValue() const
    {
  		... //derived classes不重新定义它
        int retVal = doHealthValue(); //做一些事情工作，详下
        ... //做一些事后工作
        return retVal;
	}
private:
	virtual int doHealthValue() const //derived classes可重新定义它
    {
  		... //缺省算法，计算健康指数
	}	
};
```

令客户通过public non-virtual成员函数间接调用private virtual函数，称为non-virtual interface手法(NVI)。它是所谓Template Method设计模式的一个独特表现形式。我把这个non-virtual函数称为virtual函数的wrapper。

优点：可以使得virtual函数在调用的时候，之前可以设定好适当场景，并在调用结束后清理场景。事前工作可以是锁定互斥器、制造运转日志记录项目、验证class约束条件。事后工作可以包括互斥器解除锁定，验证函数的时候条件、再次验证class约束条件等等。

### 方案3-借由Function Pointers实现Strategy模式

```c++
class GameCharacter
{
public:
	explict GameCharacter(HealthCalcFunc hcf = defaultHealthCalc);
  	void setHealthCalculator(HealthCalcFunc hcf);
private:
	HealthCalcFunc healthFunc;
};
```

优点：可以在运行时改变计算血量的方式，比如有的角色开始时计算血量是一种方式，过了某个血量范围后，又是另外一个方式。

缺点：如果计算血量，需要依赖non-public的信息，就无法进行下去。



### 方案4-借由tr1::function完成strategy模式

tr1::function对象可以使用任何可调用无，也就是函数指针、函数对象或成员函数指针。

```c++
class GameCharacter
{
public:
	typedef std::tr1::function<int (const GameCharacter &)> HealthCalcFunc;
private:
    HealthCalcFunc healthCalcFunc;
};
```

优点：灵活性，可以使用任何可调用的对象。

### 方案5-古典的strategy模式

```
          ---------------             ----------------
          |GameCharacter|/\-----------|HealthCalcFunc|
          ---------------\/            ---------------
                /\                            /\   
                ||                            ||
         -------||----------                  ||------------
         |      ||         |                  ||		   |
    ------------     --------------     -----------  -----------
    |EvilBadGuy|     |EyeCharacter|     |SlowLoser|  |FastLoser|
    ------------     --------------     -----------  -----------
```

### 6.5 绝不重新定义继承而来的non-virtual函数

绝不重新定义继承而来的non-virtual函数

```c++
class B
{
public:
	void mf();
    ...
};
class D : public B
{

};
D x;
B *pb = &x;
pb->mf();//调用B::mf
D *pd = &x;
pd->mf();//调用D::mf
```

在D中重载non-virtual的B类对象中的函数，其实已经违反了每个D都是一个B的约束。即对于non-virtual的接口，继承类应该和基类行为是一致的，否则就不应该设计成non-virtual接口。

### 6.6 绝不重新定义继承而来的缺省参数值

virtual函数系动态绑定，而缺省参数值却是静态绑定。

举个例子

```c++
class Shape
{
public:
	enum ShapeColor{Red, Green, Blue};
    virtual void draw(ShapeColor color=Red) const = 0;
};
class Rectangle : public Shape
{
public:
	virtual void draw(ShapeColor color=Green) const = 0;
};
```

有以下代码

```c++
Shape *ps;
Shape *pc = new Circle;
Shape *pr = new Rectangle;
```

用pc->draw()的时候，采用的默认参数是Red，是Shape类的默认参数，而不是Derived的类的默认参数，因为默认参数是静态编译期间绑定的。

即使把Base class和Derived class设计成相同的默认参数，如果某一天要修改这个参数，得两个类都需要修改。

这个问题可以通过NVI方法来避免，把函数功能抽象成private的virtual函数，然后把缺省的默认参数移到non-virtual的public函数中。

### 6.7 通过复合塑模出has-a

复合is-a有两种含义，一种has-a含义，另一种是is-implemented-in-terms-of，根据某物实现出。

例如，我们希望基于std::list来实现一个set，可以，在一个set类里面定义一个std::list作为内部成员，来实现set。

### 6.8 避免使用private继承

private继承意味着is-implemented-in-terms-of（根据某物实现出）。它通常是比复合（composition）的级别低。不推荐使用。

### 6.9 避免使用多重继承

- 多重继承比单一继承复杂。它可能导致新的歧义性，以及对virtual继承的需要。
- virtual继承会增加大小、速度、初始化（及赋值）复杂度等等成本。如果virtual base classes不带任何数据，将是最具实用价值的情况。
- 多重继承的确有正当用途。其中一个情节涉及“public继承某个Interface class”和“private继承某个协助实现的class”的两相组合。

## 7 模板与泛型编程

### 7.1 了解隐式接口和编译期多态

对于template编程，隐式接口和编译期多态移到了前头，例如

```c++
template<typename T>
void doProcessing(T &W)
{
  if (w.size() > 10 && w != someNastyWidget)
  {
  	T temp(w);
    temp.normalize();
    temp.swap(w);
  }
};
```

- w必须支持哪一种接口，系由template中执行于w身上的操作来决定。本例看来w的类型T好像必须支持size，normalize和swap成员函数、copy构造函数、不等比较。
- 凡涉及w的任何函数调用，例如operator>和operator!=，有可能造成template具现化，使得这些调用得以成功。这样的具现行为发生在编译期。“以不同的template参数具现化function templates”会导致调用不同的函数，这是编译期多态。

### 7.2 typename和class的区别

```c++
template<typename C>               //允许使用typename或class
void f (const C &container,        //不允许使用typename
		typename C::iterator iter) //一定要使用typename
```

因为C::iterator看起来有可能是类型，也有可能是成员变量，所以，要明确告诉编译器。

### 7.3 学习处理模板化基类内的名称

```c++
class MsgInfo{...};
template<typename Company>
class MsgSender
{
public:
	...
    void sendClear(const MsgInfo &info)
    {
  		std::string msg;
        Company c;
        c.sendCleartext(msg);
    }
    void sendSecret(const MsgInfo &info) {...}
};

template<typename Company>
class LoggingMsgSender : public MsgSender<Company>
{
public:
	...
    void sendClearMsg(const MsgInfo &info)
    {
  		将"传送前"的信息写至log;
        sendClear(info);  //调用base class函数：这段代码无法通过编译
        将"传送后"的信息写至log;
	}
}
```

解决方案有三个：

- 在sendClear之前加上this操作符

```
this->sendClear(info);
```

- 使用using声明式

```
using MsgSender<Company>::sendClear;
```

- 明白指出被调用的函数位于base class内

```
MsgSender<Company>::sendClear(info);
```

### 7.4 将与参数无关的代码抽离

对于template class或者template function，其间的代码重复可能不是很容易能看出来，需要对调用情况进行分析，可能会实现仅仅是参数不同的函数，例如

```
template<typename T,
		 std::size_t n>
class SquareMatrix
{
public:
    void invert();
};

SquareMatrix<double, 5> sm1; //调用SquareMatrix<double, 5>::invert
sm1.invert();
SquareMatrix<double, 10> sm2; //调用SquareMatrix<double, 10>::invert
sm2.invert();
```

解决方案是写一个带参数的invert函数，然后，public函数的接口调用带参数的invert函数。

```
template<typename T>
class SquareMatrixBase
{
protected:
	void invert(std::size_t matrixSize);
};
template<typename T, std::size_t n>
class SquareMatrix : private SquareMatrixBase<T>
{
private:
	using SquareMatrixBase<T>::invert;
public:
	void invert() {this->invert(n);}
};
```

### 7.5 运用成员函数模板接受所有兼容类型

```
template<typename T>
class SmartPtr
{
public:
	explicit SmartPtr(T *realPtr);
    ...
};
SmartPtr<Top> pt1 = 
	SmartPtr<middle>(new Middle); //将smartPtr<middle>转换成SmartPtr<Top>
SmartPtr<Top> pt2 = 
	SmartPtr<Bottom>(new Bottom);
SmartPtr<const Top> pct2 = pt1;
```

解决方案

```c++
template<typename T>
class SmartPtr
{
public:
	template<typename U>
   	SmartPtr(const SmartPtr<U> &other)
    	: heldPtr(other.get()) {...}
    T *get() const {return heldPtr;}
private:
	T *heldPtr;
};
```

使用成员初值列来初始化SmartPtr<T>之内类型为T*的成员变量，并以类型为U\*的指针作为初值，这个行为只有当“存在某个隐式转换可将一个U\*指针转换成一个T\*指针”才能通过编译。

在class内声明泛化copy构造函数，并不会阻止编译器生成自己的copy构造函数，所以，如果你想要控制copy构造的方方面面，你必须同时声明泛化copy构造函数和正常的copy构造函数。相同规则也适用于赋值操作。

### 7.6 需要类型转换时请为模板定义非成员函数

这个点没怎么看懂。

### 7.7 使用traits classes表现类型信息

STL迭代器分类：

- Input迭代器：只能向前移动，一次一步，客户只可读取它们所指的东西，而且只能读取一次。
- Output迭代器：只能向前移动，一次一步，客户只可涂写它们所指的东西，而且只能涂写一次。
- forward迭代器：这种迭代器可以做前述两种分类能做的每件事情，而且可以读或写其所指物一次以上。
- bidirectional迭代器：它除了可以向前移动，还可以向后移动。STL的list迭代器就属于这一分类，set，multiset，map和multimap的迭代器也都是这一分类。
- random access迭代器：可以在常量时间内向前或向后跳跃任意距离。vector，deque和string提供的迭代器都是这一分类。

在STL中有advance函数，可以把迭代器按照某个距离进行移动，声明如下：

```c++
template<typename IterT, typename DistT>
void advance(IterI &iter, DistT d)
{
  if (iter is a random access iterator)
  {
    iter += d;
  }
  else
  {
    if (d >= 0) {while(d--) ++iter;}
    else {while (d++) --iter;}
  }
}
```

c++中实现相关原理如下：

```
template <typename IterI>
struct iterator_traits;

template<...>
class deque
{
public:
	class iterator
    {
  	public:
    	typedef random_access_iterator_tag iterator_category;
    }
};

template<...>
class list
{
public:
	class iterator
    {
  	public:
    	typedef bidirectional_iterator_tag iterator_category;
	};
};

对于iterator_traits
template<typename IterT>
struct iterator_traits
{
	typedef typename IterT::iterator_category iterator_category;
};

对于内置指针，需要特化一个版本出来
template<typename IterT>
struct iterator_traits<IterT *>
{
  typedef random_access_iterator_tag iterator_category;
};

template<typename IterT, typename DistT>
void advance(IterT &iter, DistT d)
{
  if (typeid(typename std::iterator_traits<IterI>::iterator_category) == typeid(std::random_access_iterator_tag))
}; //这个在编译期就能确定的事情，没必要在运行期来确定，可以使用下面的方法


template<typename IterT, typename DistT>
void doAdvance(IterT &iter, DistT d,
				std::random_access_iterator_tag)
{
  iter += d;
}

template<typename IterT, typename DistT>
void doAdvance(IterT &iter, DistT d, std::bidirectional_iterator_tag)
{
  if (d >= 0) {while (d--) ++iter;}
  else {while(d++) --iter;}
}
template<typename IterT, typename DistT>
void doAdvance(IterT &iter, DistT d, std::input_iterator_tag)
{
  if (d < 0)
  {
      throw std::out_of_range("Negative distance");
  }
  while(d--) ++iter;
}

template<typename IterT, typename DistT>
void advance(IterT &iter, DistT d)
{
  doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}
```

- 建立一组重载函数或函数模板，彼此间的差异只在于各自的traits参数。
- 建立一个控制函数或函数模板，它调用上述那些劳工函数并传递traits class所提供的信息。

### 7.8 认识template元编程

不太懂

## 8 定制new和delete

这块后面看了C++ primer的allocator一章再看

## 9 杂项讨论

### 9.1 不要忽略编译器的警告

编译器警告报出的很可能是潜在的程序bug，需要重视。

### 9.2 让自己熟悉包括TR1在内的标准程序库

- STL
- Iostreams
- 国际化支持
- 数值处理
- 异常阶层体系
- C89标准程序库

上面的分类应该是比较老的分类了，最新的应该包含C++11和C++14的标准程序库了。

### 9.3 让自己熟悉Boost

Boost是否需要熟悉？看google代码规范是推荐使用其中一部分的，觉得还是有必要看看。
