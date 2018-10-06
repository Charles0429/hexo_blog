---
title: golang reflection学习笔记
date: 2016-09-06 20:15:19
categories: 语言
tags:
  - golang
  - reflection
---

# Introduction

reflection是golang中用来获取interface的type, value和method的方式，本文主要介绍这三种使用方式。

# golang reflection

## interface的类型

- 获取interface的type

```
t := reflect.Typeof(3)
fmt.Println(t) //"int"
```

Typeof用来获取某个interface的类型。

## interface的值

- 获取interface的value

```go
v := reflect.ValueOf(3)
fmt.Println(v) // "3"
fmt.Println("%v\n", v) // "3"
fmt.Println(v.String()) // "<int 3>"
```

ValueOf中返回值类型是reflect.Value，也可以通过它来获取其类型。

```go
v := reflect.Value(3) // a reflect.Value
t := v.Type() // 
x := v.Interface() // an interface{} 
i := x.(int) // an int
fmt.Println("%d\n", i) // "3"
```

Type()返回的是reflect.Type类型，而Interface返回的是reflect.Interface


```go
v := reflect.Value(3)
switch v.Kind() {
case reflect.Invalid:

case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:

case reflect.Uint, reflect.Uint8, reflect.Uint16, relect.Uint32, reflect.Uint64:

case reflect.String:

case reflect.Ptr:

case reflect.Array, reflect.Slice:

case reflect.Struct:
	for i:=0 ; i < v.NumFields(); i++ {
    
    }
    
case reflect.Map:

}
```

如上，Kind可以用来获取具体的类型，通过switch语句来匹配具体的类型。

- 设置reflect.Value的值

```go
x := 2
d := reflect.ValueOf(&x).Elem() // d refers to the variable x
px := d.Addr().Interface.(*int) // px := &x
*px = 3
fmt.Println(x)
d.Set(reflect.ValueOf(4))
```

可以通过ValueOf.Elem获取元素的值，然后通过Set方法设置。

## interface的函数

- 遍历struct的method

```go
v := reflect.ValueOf(x)
t := v.Type()

for i := 0; i < v.NumMethod(); i++ {
	methodType := v.Method(i).Type()
}
```

通过NumMethod来遍历，获取到对应的Method，其类型是`reflect.Value`，通过其Call方法来调用该函数。

- 通过函数名来调用

```go
v := reflect.ValueOf(x)
v.MethodByName("test").Call(v)
```

通过MethodByName获取到函数然后调用Call也能完成调用。

## 例子

下面以一个完整的例子，来完整的展示reflect.Type,reflect.Value和reflect.Method的用法。

```go
package main

import (
	"fmt"
	"reflect"
)

type Test struct {
	i int
	f float64
	m map[string]int
	v []string
}

func NewTest() Test {
	var t Test
	t.m = make(map[string]int)
	return t
}

func (t *Test) SetI(i int) {
	t.i = i
}

func (t *Test) SetF(f float64) {
	t.f = f
}

func (t *Test) SetM(key string, value int) {
	t.m[key] = value
}

func (t *Test) SetV(value string) {
	t.v = append(t.v, value)
}

func main() {
	t := NewTest()
	fmt.Println(reflect.TypeOf(t)) //main.Test
	v := reflect.ValueOf(&t)
	fmt.Println(v.Kind()) // ptr
	typ := reflect.TypeOf(t)
	fmt.Println(v.Type()) // *main.Test
	for i := 0; i < typ.NumMethod(); i++ {
		method := typ.Method(i)
		fmt.Println("%s", method.Name)
	}
	fmt.Println(t) // {0 0 map[] []}
	args := make([]reflect.Value, 0)
	args = append(args, reflect.ValueOf(2.16))
	v.MethodByName("SetF").Call(args)
	fmt.Println(t) // {0 2.16 map[] []}
}
```
PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://oserror.com/images/qcode_wechat.jpg)

