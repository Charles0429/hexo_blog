---
title: The go programming language 学习笔记
date: 2016-06-16 08:09:23
categories: 语言
tags:
  - golang
---

# 1. Introduction

本文是学习The go programming language的总结，全文的组织结构如下：

- Program Structure
- Basic Data Types
- todo


# 2. Program Structure

## 2.1 Names

- 合法的命令为以字母为下划线开头，并且除关键字外的字符串（GO对大小写敏感）
- 如果一个命令定义在函数外面，并且它的首字母是大写的，那么它可以被package外的代码访问到；如果定义在函数外部，但首字母是小写，那么能被package内部访问到
- GO推荐用驼峰命名法来命名，即parseRequestLine是推荐的，但parse\_request\_line是不推荐的
- 缩写的名称，一般全用大写，例如ASCII、HTML等

## 2.2 Declarations

以一个例子来说明

```go
package main

import "fmt"

const boilingF = 212.0

func main() {
	var f = boilingF;
    var c = (f - 32) * 5 / 9;
    fmt.Printf("boiling point = %g F or %g C", f, c);
}
```
- 常量boilingF是package级别的定义
- 变量f和c是本地变量，属于函数main内部
- 函数的定义为函数名+参数列表+返回值，main函数把后两者省略掉了

## 2.3 Variables

变量的声明形式为

```go
var name type = expression
```

- 如果type被省略，那么根据expression字段的类型推断出来
- 如果expression被省略，那么初始化成zero value,对于numbers为0，对于boolean为false，对于strings为空，对于interface和reference type(slice，pointer，map，channel，function）为nil。

可以在一条语句中声明多个变量

```go
var i, j, k int
var b, f, s = true, 2.3, "four
```

### 2.3.1 Short Variable Declarations

```go
name := expression
```

会以expression的type来推断name的type，并且以expression初始化name

大部分的局部变量会用Short Variable Declaration方式来定义，两种情况除外

- 声明的变量的类型和初始化的expression不一致，用var显式地声明
- 声明的时候不需要初始化，后续会赋值

`:=`和`=`的区别是，前者是声明，后者是赋值

在`:=`语句中，不用声明所有的变量，如果有变量已经声明过，就相当于转成了赋值操作，但是，必须至少有一个变量没有声明过，因为`:=`语句要求至少新声明一个变量。

```go
//声明了out变量，err是赋值
in ,err := os.Open(infile)
out, err := os.Create(outfile)

//编译错误，第二个语句没有新声明任何的变量
f, err := os.Open(infile)
f, err := os.Create(outfile)
```

### 2.3.2 Pointers

pointer存储的是变量的地址，可以通过pointer来间接的更新变量的值。

**简单例子**

```go
x := 1
p := &x
fmt.Println(*p) //1
*p = 2
fmt.Println(x)  //2
```

**局部变量返回成指针也是安全的**

```go
var p = f()

func f() *int {
	v := 1
    return &v
}
```

和C不一样，在GO中这是合法的，v在函数返回后继续存在。

### 2.3.3 The *new* function

new返回的也是指针，例如
```go
p := new(int)
fmt.Println(*p)
*p = 2
fmt.Println(*p)
```

下面两个函数是等价的，即new和普通的局部变量区别不大

```go
func newInt() *int {
	return new(int)
}

func newInt() *int {
	var dummpy
    return &dummpy
}
```

一般来说，new创建出来的变量地址是不同的，但是，有可能struct {} 这样的空结构体，地址是相同的，取决于编译器的实现。

### 2.3.4 Lifetime of Variables

- package级别的变量，生命周期是整个程序运行期间
- 局部变量是在声明语句开始后，到变量已经无法使用（即没有符号引用它，包括变量名或者指针）

局部变量既可能在heap上创建，也可能在stack上创建，和是否用new无关

```go
var global *int

func f() {
	var x int
    x = 1
    global = &x
}

func g() {
	y := new(int)
    *y = 1
}
```
x是在heap上创建的，因为一直被global引用，而y是stack创建，因为出了函数后，就没有使用了。

GO的垃圾回收就是通过判断变量是否还被符号引用来做的，如果没有符号引用了，即表明可以回收这块的内存空间。

## 2.4 Assignment

赋值语句会让变量的值更新，例如

```go
x = 1
*p = true
person.name = "bob"
count[x] = count[x] * scale
```

### 2.4.1 Tuple Assignment

元组赋值例子如下

```go
x, y = y, x
```

有些操作，会返回多个值，可以使用元组操作，例如

```go
v, ok = m[key]
v, ok = x.(T)
v, ok = <-ch
```

### 2.4.2 Assignability

赋值操作的左右类型必须是一致的；nil可用于interface和reference type的赋值

## 2.5 Type Declarations

定义type的形式如下
```go
type name underlying-type
```

type和underlying-type之间是不能直接赋值的，因为它们不是相同的类型。

比较运算符可以用来比较相同的type，或者type和underlying-type，但是不能用于比较named type，例如

```go
type Celsius float64
type Fahrenheit float64
var c Celsius
var f Fahrenheit
c == 0 //true
f >=0 //true
c == f //compile error
```
c和f不能直接比较，两个都属于named type。

## 2.6 Package and Files

每个package相当于独立的命名空间，一般在同一个目录下的一个或多个文件可以组成一个package。

例如，对于package `gopl.io/ch1/helloworld`对应的path是`$GOPATH/src/gopl.io/ch1/helloworld`。

以一个例子来说明，例如，我们需要建立一个温度转换的package，名字为`gopl.io/ch2/tempconv`

有两个文件，分别是`tempconv.go`和`conv.go`

```go
package tempconv

type Celsius float64
type Fahrenheit float64

const (
	AbsoluteZeroC Celsius = -273.15
    FreezingC Celsius = 0
    BoilingC Celsius = 100
)

func (C Celsius) String() string {
	fmt.Sprintf("%gC", c)
}

func (f Fahrenheit) String() string {
	fmt.Printf("%gF", f)
}
```

```go
package tempconv

func CToF(c Celsius) Fahrenheit {
	return Fahrenheit(c * 9 / 5 + 32)
}

func FToC(f Fahrenheit) Celsius {
	return Celsius((f - 32) * 5 / 9)
}
```

package中以大写开头的全局变量或函数都能被外部调用到。

### 2.6.1 Imports

以import语句来导入package，例如

```go
import (
	"fmt"
    "os"
    "strconv"
    "gopl.io/ch2/tempconv"
)
```

在GO里面，如果导入一个package，但是没有引用，会报编译错误。

有`golang.org/x/tools/cmd/goimports`工具，可以自动的插入和删除需要的package。

### 2.6.2 Package Initialization

package初始化会按照变量声明的顺序初始化。如果对于一些比较复杂的数据结构，可能仅仅通过初始化语句无法完成初始化，这时候可以把初始化操作放到init函数中，init函数会自动的被执行。

package初始化之前，会先把要导入的package初始化。对于package main会在最后初始化，可以保证在main函数执行之前，其他package已经完成初始化了。

## 2.7 Scope

Scope代表变量的声明在程序哪个位置，而lifetime则表示变量在程序执行的可以被引用的时间段。前者是编译时期的特性，而后者是运行时的特性。

Scope一般包括universe，package，file和function。

- 内置类型，函数，常量为universe level
- 定义在函数外，可以被相同package的任意file引用，称为package level
- 通过文件中import过，例如 import fmt，则fmt的函数在本文件都是可用的，称为file level
- 函数内部的定义，只有函数内部应用，则为function level

和C一样，范围越小的变量会隐藏范围大的变量。

**for语句**

for定义了两个block

- 显式的block，用{}包起来
- 隐式的block，如初始化中的变量，范围是for循环条件，自增以及显式的block内

**if语句**

```go
if x := f(); x == 0 {
	fmt.Println(x)
} else if y := g(x); x == y {
	fmt.Println(x, y)
} else {
	fmt.Println(x, y)
}
fmt.Println(x, y) //compile error: x和y在这里不可见


if f, err := os.Open(fname),; err != nil { //complie error: unused f
	return err
}

f.ReadByte() //compile error: undefined f
f.Close() // compile error: undefined f
```

# 3. Basic Data Types

GO的Data Type有四大类：basic types，aggregate types，reference types和interface types。Basic types包括numbers，strings和booleans；Aggregate types包括array和struct；Reference types包括pointers，slices，maps，functions和channels。

## 3.1 Integers

GO的numbers类型包括integers，floating-point numbers和complex numbers。

对于integers，有四种有符号整数和四种无符号整数。

- int8, int16, int32和int64
- uint8, uint16, uint32和uint64

除了带字节大小的类型之外，还包括int和uint，可能是32bit或者64bit，由编译器决定，编程时不要假定这个大小。

rune是int32的named type，用来存储单个unicode字符。

byte是int8的named type。

uintptr的大小可以存储下系统中任意的内存地址，一般用来和C Library交互的时候。

采用二进制补码的方式编码，所以对于int8来讲，其范围为[-128, 127]

操作符的优先级如下

```go
*	/	%	<<	>>	&	&^
+	-	|	^
==	！=	<	<=	>	>=
&&
||
```
对于%操作符，符号是跟着被除数走的，例如`-5%3`和`-5%-3`的余数都是-2。

对于算术运算，可能会溢出

```go
var u uint8 = 255
fmt.Println(u, u+1, u*u) //255 0 1
```

对于移位操作符

- `<<`不管是有符号数或者无符号数，都是末尾补0
- `>>`对于有符号数，会在左边补符号位，对于无符号数，会在末尾补0

对于遍历操作，一般用有符号数

```go
var i uint = 0
for ; i >= 0; i-- {

}
```

上面是个死循环，因为uint一定是大于或等于0的，故遍历的索引一般用有符号数。

## 3.2 Floating-Point Numbers

GO提供两种float类型，即float32和float64，采用IEEE 754标准。

## 3.3 Complex Numbers

GO提供两种complex类型，即complex64和complex128，底层组件分别用float32和float64。

使用例子如下

```go
var x complex128 = complex(1, 2)  //1 + 2i
var y complex128 = complex(3, 4)  //3 + 4i
fmt.Println(x*y)                  //-5 + 10i
fmt.Println(real(x*y))			  //-5
fmt.Println(imag)(x*y))			  //10
```
## 3.4 Booleans

bool类型的值只有false和true两种。bool类型和number类型之间没有隐式的转换，一般通过如下方式

```go
func btoi(b bool) int {
	if b {
    	return 1
    }
    return 0
}

func itob(i int) bool {
	return i != 0
}
```

## 3.5 Strings

内置len函数可以计算string类型的长度。

substring操作通过s[i:j]，表示从i开始，共j-i个字符，因此，不包括j。

i的默认值为0，j的默认值为len(s)，因此

```go
s[:5] = s[0:5]
s[7:] = s[7:len(s)]
s[:] = s[0:len(s)]
```
string类型是不可修改的，例如

```
s = "hello"
s += “world”
s[0] = 'L' // compile error
```
s开始指向"hello"字符串，执行`+=`操作后，并不是修改原来字符串变成"hello world"，而是完全分配新的内存空间，把"hello world"存进去，然后修改s指向这块内存空间。

因为string的不可修改，所以substring可以和原来的string共享内存空间。

### 3.5.1 String Literals

- 用双引号，例如"hello world"
- 用\来做转义

GO还提供\`...\`来作为raw string literal的声明，即里面的转义字符像\，换行都不会特殊处理，所以，可以放到多行。一般可以放\等特别多的字符串。

### 3.5.2 Unicode

Unicode是为了解决各国文字无法在ASCII表示出来的困境，Unicode也分为多种

- UTF-32，每个Unicode字符都采用32位存储
- UTF-8，每个Unicode字符的存储空间不定，采用前缀的方式来区分

### 3.5.3 UTF-8

UTF-32的缺点有

- 对于普通的ASCII也要采用32位存储，不兼容
- 对于常用的65536个Unicode字符，其实用16位就行，32位会浪费大量的存储空间

GO里面有专门的package`unicode/utf8`来处理UTF-8格式的编解码等。

## 3.6 Constants

const类型的语义是在运行期间，变量的值不会发生变化。const可用于boolean，string和number。

### 3.6.1 The Constant Genrator itoa

和C不同的是，const默认的是和上一个值相同，例如

```go
const (
    a = 1
    b
    c
)
```
此时，b和c都是为1。

GO里边提供itoa来实现C中enum值自增的方法，如下

```go
type Weekday int
const (
    Sunday Weekday = itoa
    MOnday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```

上面定义中，Sunday为0，Monday为1，以此类推。

还有如下用法

```go
type Flags uint

const (
    FlagUp flags = 1 << itoa
    FlagBroadcast
    FlagLoopback
    FlagPointToPoint
    FlagMulticast
)
```
当itoa递增时，每个const会赋值成`1 << itoa`对应的值。

更有趣的有

```go
const (
	_ = 1 << (10 * itoa)
    KiB //1024
    MiB //1048576
    GiB //1073741824
    TiB
    PiB
    EiB
    ZiB
    YiB
)
```

### 3.6.2 Untyped Constants

untyped const可以不绑定到特定的类型，这样的const一般至少有256位的精度，所以，可以参与更高精度的计算。在赋值的时候，untyped const会隐式的转换到对应的类型，例如

```go
var x float32 = math.Pi //untyped const
var y float64 = math.Pi
var z complex128 = math.Pi

const Pi64 float64 = math.Pi
var x float32 = float32(Pi64) //需要转类型转成，因为不是untyped const
var y float64 = Pi64
var z complex128 = complex128(Pi64)
```
共有六种类型的untyped const，分别是untyped boolean，untyped integer，untyped rune，untyped rune，untyped floating-point，untyped complex和untyped string。

例如，true和false是untyped boolean，字符串常量是untyped string。

```go
var f float64 = 3 + 0i  //untyped complex -> float64
f = 2                   //untyped integer -> float64
f = 1e123				//untyped floating-point -> float64
f = 'a'					//untyped rune -> float64
```
这种隐式的转换需要左边的变量能表示右边的值，有些情况是不能转换的，如下

```go
const (
    deadbeef = 0Xdeadbeef
    a = uint32(deadbeef) // uint32 with value 3735928559
    b = float32(deadbeef) // float32 with value 
    d = int32(deadbeef) // compile error: overflow
    e = float64(1e309) //compile error: const overflows float64
    f = uint(-1) //compile error: const underflows uint
)
```

在变量声明中，如果没有指定类型，会由untyped const来隐式地决定类型

```go
i := 0  //integer
r := '\000' //rune
f := 0.0 //float64
c := 0i //complex128
```