---
layout: post
title: Go 反射的实现（二）
description: 从底层代码介绍 Go 如何实现反射，并总结出一些思考。
use_mermaid: true
---

在去年曾写过一篇文章 [Go 反射原理的实现]({{ "/2022/09/10/go-reflection.html" | relative_url }})，
但是在一年时间内，对是否用反射，怎么用反射，为什么要用反射，反射是怎么做的又有了新的思考。

# Why，为什么要用反射

我们程序的本质是数据+代码，代码是控制数据流动的。控制数据流动之前有个重要的前提是，知道数据的位置。数据的位置包括数据的起始位置和长度。起始位置可以用
编译器帮我们搞掂，不需要关心，而长度和数据类型强相关。这就引入了我们的主题，反射起始就是为了获取数据的类型。举一个最简单的例子，一个加法函数：

```go
package main

func add(a, b int32) int32 {
	return a+b
}

func main() {
	_ = add(1, 2)
}
```

我们把它生成汇编代码后如下：

```text
main.add STEXT nosplit size=64 args=0x8 locals=0x18 funcid=0x0 align=0x0 leaf
	0x0000 00000 (main.go:3)	TEXT	main.add(SB), NOSPLIT|LEAF|ABIInternal, $32-8
	...
	0x0018 00024 (main.go:4)	MOVW	main.b+4(FP), R1   # 从 FP+4 读取 b 到 R1
	0x001c 00028 (main.go:4)	MOVW	main.a(FP), R2     # 从 FP+0 读取 a 到 R2
	0x0020 00032 (main.go:4)	ADD	R1, R2, R0             # 寄存器 R1+R2 并把结果放到 R0
	0x0024 00036 (main.go:4)	MOVW	R0, main.~r0-4(SP) # 把寄存器 R0 结果放到 SP $0-4
	...
```

这个例子展示了程序是如何完成 1+1 这样的加法操作，FP 其实是确定了数据的内存起始地址，MOVW 确定了数据的大小。

| directive | full name        | description |
|-----------|------------------|-------------|
| MOVB      | move byte        | 移动 1 bytes  |
| MOVH      | move ?           | 移动 2 bytes  |
| MOVW      | move word        | 移动 4 bytes  |
| MOVD      | move double word | 移动 8 bytes  |

## 泛型和反射的区别

我们可以通过汇编代码来看。实现一个泛型的加法函数：

```go
package main

func add[T int8|int16|int32|int64|int](a, b T) T {
	return a+b
}

func main() {
	_ = add(int8(1), int8(2))
	_ = add(int16(1), int16(2))
	_ = add(int32(1), int32(2))
	_ = add(int64(1), int64(2))
}
```

生成汇编代码的结果是：

```text
main.add[go.shape.int8_0] STEXT dupok nosplit size=64 args=0x10 locals=0x18 funcid=0x0 align=0x0 leaf
	0x0000 00000 (main.go:3)	TEXT	main.add[go.shape.int8_0](SB), DUPOK|NOSPLIT|LEAF|ABIInternal, $32-16
	...
	0x001c 00028 (main.go:4)	MOVB	main.b+9(FP), R1
	0x0020 00032 (main.go:4)	MOVB	main.a+8(FP), R2
	0x0024 00036 (main.go:4)	ADD	R1, R2, R0
	0x0028 00040 (main.go:4)	MOVB	R0, main.~r0-1(SP)
    ...
main.add[go.shape.int16_0] STEXT dupok nosplit size=64 args=0x10 locals=0x18 funcid=0x0 align=0x0 leaf
	0x0000 00000 (main.go:3)	TEXT	main.add[go.shape.int16_0](SB), DUPOK|NOSPLIT|LEAF|ABIInternal, $32-16
    ...
	0x001c 00028 (main.go:4)	MOVH	main.b+10(FP), R1
	0x0020 00032 (main.go:4)	MOVH	main.a+8(FP), R2
	0x0024 00036 (main.go:4)	ADD	R1, R2, R0
	0x0028 00040 (main.go:4)	MOVH	R0, main.~r0-2(SP)
	...
main.add[go.shape.int32_0] STEXT dupok nosplit size=64 args=0x10 locals=0x18 funcid=0x0 align=0x0 leaf
	0x0000 00000 (main.go:3)	TEXT	main.add[go.shape.int32_0](SB), DUPOK|NOSPLIT|LEAF|ABIInternal, $32-16
	...
	0x001c 00028 (main.go:4)	MOVW	main.b+12(FP), R1
	0x0020 00032 (main.go:4)	MOVW	main.a+8(FP), R2
	0x0024 00036 (main.go:4)	ADD	R1, R2, R0
	0x0028 00040 (main.go:4)	MOVW	R0, main.~r0-4(SP)
	...
main.add[go.shape.int64_0] STEXT dupok nosplit size=64 args=0x18 locals=0x18 funcid=0x0 align=0x0 leaf
	0x0000 00000 (main.go:3)	TEXT	main.add[go.shape.int64_0](SB), DUPOK|NOSPLIT|LEAF|ABIInternal, $32-24
	...
	0x001c 00028 (main.go:4)	MOVD	main.b+16(FP), R1
	0x0020 00032 (main.go:4)	MOVD	main.a+8(FP), R2
	0x0024 00036 (main.go:4)	ADD	R1, R2, R0
	0x0028 00040 (main.go:4)	MOVD	R0, main.~r0-8(SP)
	...
```

我们不难发现，泛型其实是编译器识别出类型后分别生成不同的函数。和反射不同的是，泛型在运行过程中和手戳两个不一样类型的函数性能几乎无差别，所以说泛型的
性能优于反射。

# Reference

- [Go 汇编概述](https://hopehook.com/post/golang_assembly)
- [go-internals](https://github.com/go-internals-cn/go-internals)
