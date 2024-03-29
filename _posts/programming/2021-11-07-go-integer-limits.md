---
layout: single
title: Computing integer limits in Golang
toc: true
tags: go
---

Integer limits and Golang specification for constant expressions

# How to calculate integer limit values?

Go provides all the maximum values for integer types inside the [**math** package](https://pkg.go.dev/math#pkg-constants)

```go
const (
	MaxInt    = 1<<(intSize-1) - 1
	MinInt    = -1 << (intSize - 1)
	MaxInt8   = 1<<7 - 1
	MinInt8   = -1 << 7
	MaxInt16  = 1<<15 - 1
	MinInt16  = -1 << 15
	MaxInt32  = 1<<31 - 1
	MinInt32  = -1 << 31
	MaxInt64  = 1<<63 - 1
	MinInt64  = -1 << 63
	MaxUint   = 1<<intSize - 1
	MaxUint8  = 1<<8 - 1
	MaxUint16 = 1<<16 - 1
	MaxUint32 = 1<<32 - 1
	MaxUint64 = 1<<64 - 1
)
```

For instance, to obtain the biggest 64-bit unsigned integer, first we compute the number immediately greater than `MaxUint64` with the expression `1<<64` (that is a 65-bit number), then we decrease it by 1.

So, I opened a [Go playground](https://play.golang.org/p/t2eJkjlJKpg) to test a line that prints the biggest 64-bit unsigned integer  `fmt.Println(1<<64 - 1)` but the compiler complains about an **overflow**: `./prog.go:8:20: constant 18446744073709551615 overflows int Go build failed.`.

How is it possible that **math** uses a 65-bit integer and my code can't compile, instead? Actually, the two expressions are quite different:
- math package assigns that expression to a **const** variable
- my code passes that expression as an argument to a function

## Golang constant expressions

Golang [specification](https://golang.org/ref/spec#Constant_expressions) clarifies this behaviour:

_Constant expressions are always evaluated exactly; intermediate values and the constants themselves may require precision significantly larger than supported by any predeclared type in the language. The following are legal declarations:_

```go
const Huge = 1 << 100         // Huge == 1267650600228229401496703205376  (untyped integer constant)
const Four int8 = Huge >> 98  // Four == 4                                (type int8)
```

## Negate instead of bit-shift

The biggest 64-bit unsigned integer has all bits set to 1. Another way to obtain the biggest 64-bit unsigned integer is to negate a **uint64** set to 0.

```go
var maxUint64 uint64 = ^uint64(0)
```
