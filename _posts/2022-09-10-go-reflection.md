---
layout: post
title: Go 反射原理的实现
description: 从底层代码介绍 Go 如何实现反射，并总结出一些思考。
use_mermaid: true
---
# 什么是反射

首先要明确一点，程序的本质是代码 + 数据，我们编写代码的本质是为了控制数据、处理数据。编译其实就是把高级语言转换成机器码，一堆明确的 CPU 操作指令。
比如一个函数：`(x, y) ⇒ x+y`，他就是控制 CPU 从内存读取数据到寄存器，CPU 运算后再从寄存器写入内存这样一个过程。这个过程我们在编译过程中就能确定
他的类型，知道内存地址。但是在现代编程中，我们常常会有动态的东西，运行时才知道操作的数据是什么，无法编译时候确定，就需要反射。比如最常用的 json 序列
化场景，我们直接 `json.Marshal` 就完事了，但是它是怎么实现的呢？其实使用的是反射（[链接](https://github.com/golang/go/blob/master/src/encoding/json/encode.go)）：

```go
func Marshal(v any) {
	reflectValue(reflect.ValueOf())
}

func reflectValue(v reflect.Value) {
	valueEncoder(v)()
}

// 获取当前 value 对应类型的 encoder
func valueEncoder(v reflect.Value) encoderFunc {
	return typeEncoder(v.Type())
}

// 根据类型，返回不同的 encoder。会使用 reflect.Type 作为 key 来缓存 encoderFunc
func typeEncoder(t reflect.Type) encoderFunc {
	f, loaded = encoderCache.LoadOrStore()
	if loaded {
		return f
	}
	f = newTypeEncoder(t)
	encoderCache.Store()
	return f
}

func newTypeEncoder(t reflect.Type) encoderFunc {
	switch t.Type() {
	case reflect.Bool:
		...
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		...
	case reflect.Struct:
		...
	...
	}
}
```

`json.Marshal` 通过判断输入类型的 `reflect.Type` 来使用不同的 encoder，从而 "动态" 地选择编码方式。

像这种 JSON 序列化的场景是不是非用反射不可？那也不一定，因为使用反射本质是编译的时候不能够确定字段的类型，struct 里面有什么字段，但如果编译时候能
确定也是可以的，比如 zap 库实现的 [json encoder](https://github.com/uber-go/zap/blob/master/zapcore/json_encoder.go)，在我们追
加字段的时候就指定类型，在我们追加 struct 的时候实现 `zapcore.ObjectMarshaler`，从而避免反射。

另一种思路也类似的，如 [ffjson](https://github.com/pquerna/ffjson) 和 [easyjson](https://github.com/mailru/easyjson)，他们通过
生成代码的方法预先写好代码，可以简单地想象成生成的代码就是字符串拼接逻辑。

总的来说，反射就是运行时获取到数据的类型结构等类型信息，并加以操作。很多无法编译时确定的逻辑都可以使用反射来实现。除此之外可以把一些逻辑使用反射进行
简化，避免复杂的类型判断，当然这也需要和性能做权衡。

# 反射是怎么做的

## 背景知识点

### eface 和 iface

首先要理解 eface 和 iface。Go 里面的 interface 有两种形态，分别是 eface 和 iface，简单来说，当我们使用这个 interface 类型的变量不需要它
的方法集合时，他就是 eface，否则是 iface。比如：

```go
type MyInterface interface {
	Hello(word any)
}

type MyStruct {
}

func (MyStruct) Hello(word any) {
	fmt.Println(word)
}

func mian() {
	var a MyInterface = &MyStruct{}
	b := 1
	a.Hello(b)
}
```

在这个例子中，我们需要用到 a 对应 interface 的方法集，即 MyInterface，它的类型是 iface，而 b 仅作为一个值传递，word 是 eface。

### unsafe.Pointer 类型转换

unsafe.Pointer 是忽略编译器类型检查的类型强制转换。看一段代码，[在线运行](https://go.dev/play/p/RdlYLEXGyWA)。

```go
package main

import (
	"fmt"
	"unsafe"
)

type One struct {
	Field1 int64
}

type Two struct {
	Field1 int32
	Field2 int32
}

func main() {
	one := One{520<<32 | 10086}
	// 通过 unsafe.Pointer 强制把 *One 转换成 *Two。他们结构不一样，类型只是内存数据的表现形式。
	fmt.Println(one)
	two := *(*Two)(unsafe.Pointer(&one))
	fmt.Println(two)
}
```

如果写惯 C/C++ 的朋友对这种代码应该不陌生，其实我们类型（对象）的本质是我们程序里的以我们知道的方式去读取内存，指针只是一个内存地址，对于一段内存
我们可以通过任意方式去读取（强制转换）。

![内存数据转换及读取示意]({{ "/assets/images/2022-09-10-C++-memory-description.excalidraw.png" | relative_url }})

如上图，同一段数据使用不同类型的指针去获取能有不一样的结果。而 Go 里的反射，其实就是加了类型检查的指针数据获取。

这里有个问题，Go 不是没有对象头吗？怎么检查的结果。是的，Go 没有对象头，他对数据的检查是在编译阶段，他是强类型的，除非你使用 `unsafe.Pointer`。
另一种检查类型发生在 interface{} 进行断言，比如我们熟悉的 `a.(b)` 这种场景，因为 interface{} 持有 _type，相当于它的对象头。下文会展开介绍
_type 的结构。

### struct 的内存对齐

由于硬件限制，内存只能读取整个单元内的数据，即位宽，如 64bit。而且 CPU 的 Cache Line 也是对齐后的数据进行缓存。在操作系统层面，提供给我们的指令
是指令总是对齐地读取内存。下面是一个字段对齐后的例子（[在线运行](https://go.dev/play/p/jndGsJdepx6)）（例子是 64 位系统）：

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	a := struct {
		i1 int8
		i2 int64
		i3 int32
		i4 int16
		i5 int8
		i6 int8
		i7 int8
	}{}
	fmt.Printf("Sizeof(a) = %d\n", unsafe.Sizeof(a))         // 32
	fmt.Printf("Offsetof(i1) = %d\n", unsafe.Offsetof(a.i1)) // 0
	fmt.Printf("Offsetof(i2) = %d\n", unsafe.Offsetof(a.i2)) // 8
	fmt.Printf("Offsetof(i3) = %d\n", unsafe.Offsetof(a.i3)) // 16
	fmt.Printf("Offsetof(i4) = %d\n", unsafe.Offsetof(a.i4)) // 20
	fmt.Printf("Offsetof(i5) = %d\n", unsafe.Offsetof(a.i5)) // 22
	fmt.Printf("Offsetof(i6) = %d\n", unsafe.Offsetof(a.i6)) // 23
	fmt.Printf("Offsetof(i7) = %d\n", unsafe.Offsetof(a.i7)) // 24
}
```

它在内存中的布局大概是酱紫的：

![内存对齐内存布局示意]({{ "/assets/images/2022-09-10-struct-align.drawio.svg" | relative_url }})

这里明显看到，i7 和 i1 后面有一大堆空隙没有被使用，有个优化的办法是把 i7 的顺序放在 i1 后面，这样整个 struct 的体积能从 32B 变成 24B。

题外话：曾看到 cgo 文档有一句话提到过未来 Go 可能会做 struct 字段排序优化，如果优化了可能会导致 runtime 的数据布局和 Go 里不一致，另外有很多
程序使用了 cgo 会导致 cgo 里的内存布局和 Go 不一致，也是要考虑的问题，cgo 历史包袱太重了。

## TypeOf

说回正题，TypeOf 其实巧妙地利用了编译器会把传入数据转成 interface， 而 runtime 里 eface 会保存数据的类型信息这个特性从而获取他的类型。在
runtime 里 [eface](https://github.com/golang/go/blob/5c8ec89cb53025bc76b242b0d2410bf5060b697e/src/runtime/runtime2.go#L207-L210)
的结构是：

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

而 [_type](https://github.com/golang/go/blob/5c8ec89cb53025bc76b242b0d2410bf5060b697e/src/runtime/type.go#L35-L52) 的结构是：

```go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

当我们调用 `reflect.TypeOf` 时，传递的值在 runtime 会转成 interface{}，再通过使用 `unsafe.Pointer` 强转成底层的存储结构就能获取到 eface 实际在内存上存储的数据了。

```go
package reflect

func TypeOf(i any) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

我们看 reflect 包的实现，其实他就是强转成 eface，从而获取到 runtime 里 _type 的数据。

## ValueOf

`reflect.ValueOf` 和 `reflect.TypeOf` 很类似，也是通过 eface 转换来取得类型和值的指针。

这里有个特殊的处理，所有传入 `ValueOf` 的值都会逃逸到堆上，因为 map 和 chan 的生命周期处理比较复杂，所以全部放到堆上比较好处理。原话：

> Maybe allow contents of a Value to live on the stack. For now we make the contents always escape to the heap. It makes
> life easier in a few places (see chanrecv/mapassign comment below).

## 自己实现一个反射

首先我们要理解指针和 eface 在内存中的布局，一个草图：

![eface 内存布局]({{ "/assets/images/2022-09-10-eface-memory.drawio.svg" | relative_url }})

eface 由 2 个指针组成，分别指向对象类型和对象值。对象类型的原始结构在[这里](https://github.com/golang/go/blob/go1.19/src/runtime/type.go#L35-L52)。

下面是直接取 kind 和 word 的例子，[在线执行](https://go.dev/play/p/UVYb8MZUPTj)。

这个例子仅用来更好地理解反射是怎么做的，线上千万不要这么滥用 `unsafe` 包。

```go
package main

import (
	"fmt"
	"unsafe"
)

// 获取指针大小，64 位系统返回 8，32 位系统是 4
const ptrSize = unsafe.Sizeof(uintptr(0)) 

func Reflect(i any) {
	// 对当前 interface 取址
	ptr := unsafe.Pointer(&i)
	// 取 offset = 0 的值，即 typ 的值，是一个指针，即 typPtr 的值是指向 typ 类型的内存地址
	typPtr := *(*uintptr)(ptr)
  // 取 kind，offset 即为前面字段的大小之和。
  // 其实这里忽略了字段 align 的问题，但是 typ 的结构体顺序使得 32 位和 64 位系统都刚好对齐。
	kind := *(*uint8)(unsafe.Pointer(typPtr + ptrSize + ptrSize + 4 + 1 + 1 + 1))
	// 取 offset = 8 的值，即 word 的值，它的结果是指向实际存储对象的内存地址。
	value := unsafe.Pointer(*(*uintptr)(unsafe.Pointer(uintptr(ptr) + ptrSize)))

	// kind 取值是固定的，参考
	// https://github.com/golang/go/blob/go1.19/src/runtime/typekind.go
	switch kind {
	case reflect.Int:
		intVal := *(*int)(value)
		fmt.Printf("kind:int, value:%d\n", intVal)
	case 24:
		strVal := *(*string)(value)
		fmt.Printf("kind:string, value:%s\n", strVal)
	}
}

func main() {
	Reflect(1) // kind:int, value:1
	Reflect("abc") // kind:string, value:abc
}
```

上面的 `uintptr` 和 `unsafe.Pointer` 可能有点绕，需要知道一些规律即可：

1. 一个变量通过取址后类型转换能转换成 unsafe.Pointer，unsafe.Pointer 能类型转换成 uintptr。变量取址**不能**直接变成 uintptr。
2. uintptr 通过类型转换可以转成 unsafe.Pointer，unsafe.Pointer 通过类型转换能转换成变量的指针。uintptr **不能**直接转换成变量指针。
3. 指针运算（指指针地址 +1 这种），只能通过 uintptr 进行。

<div class="mermaid">
flowchart LR
    Value <-->|&| unsafe.Pointer <--> uintptr
</div>

# 怎么用反射

由上面反射的过程我们可以发现，相比起直接使用字段，反射至少会多 3  次的指针取值的操作。除此之外，一些编译优化也会因此失效，还有 GC 引用计数更复杂了。
在我们开发过程中，能不用反射的就尽量不适用反射。

一些常用的方法调用转换示意图：

<div class="mermaid">
flowchart LR
    input -->|TypeOf| Type
    input -->|ValueOf| Value
    Type -->|Kind| Kind
    Type -->|Field| StructField
    Type -->|New| Value
    Type -->|Elem| Type
    StructField -->|Name| FieldName
    StructField -->|Tag| StructTag
    StructTag -->|Find| Tag
    StructField -->|Type| Type
    Value -->|Type| Type
    Value -->|Field| Value
    Value -->|Int| int
    int -->|SetInt| Value
    Value -->|String| string
    string -->|SetString| Value
    Value -->|Elem| Value
</div>

## 获取类型

先使用 `reflect.TypeOf` 获取到 `reflect.Type`，获取到变量的类型。

## 获取值

如果需要用到变量的值才使用 `reflect.ValueOf` 获取到 `reflect.Value`，然后可以通过 `.Type()` 获取到 `reflect.Type` 进行类型判断。

## 容易踩的坑

- 获取的 StructField，如果需要使用到 `Interface()` 方法，则需要判断是否 Exported 字段，否则会 panic。
- `SetString`、`SetInt` 等设置值前需要确保类型是对应的，否则会 panic。可以用 `AssignableTo()` 进行判断。
