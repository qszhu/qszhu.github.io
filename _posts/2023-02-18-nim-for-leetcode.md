---
layout: post
title: 谈谈Nim与JavaScript的互操作——以LeetCode为例（FFI篇）
date: 2023-02-18 21:30:00 +0800
tags: [nim, leetcode, competitive_programming]
---

* **谈谈Nim与JavaScript的互操作——以LeetCode为例（FFI篇）** <-- 你在这里
* [谈谈Nim与C/C++的互操作——以OpenGL为例](/2023/12/17/nim-for-opengl.html)

Nim作为一门胶水语言，像极了当年作为一种全栈语言大杀四方的Python。然而对其特色的跨语言调用功能的描述却散落在不同的文档中，缺乏有效的整理。本文尝试整理一下Nim与JavaScript互操作的用法。

LeetCode作为一个编程竞赛网站，本身并不支持Nim，但其对JavaScript的支持比较友好（如更宽松的执行时间，以及调整到2^53之内的数据范围）。其上的题型恰好能覆盖大部分与JavaScript进行互操作的情况。所以本文就以LeetCode为例进行整理。

## 1. 输出JavaScript源代码

要在LeetCode上使用Nim，可以让Nim编译器输出JavaScript[1]：

```bash
$ nim js solution.nim
```

默认输出的代码会包含大量调试信息和运行时检查，不需要的时候可以去掉来减少代码体积[2]：

```bash
$ nim js -d:danger solution.nim
```

## 2. 导出函数

LeetCode上的题目通常是要你实现一个函数，提交后会有一些驱动代码来调用这个函数，所以需要让Nim输出一个具有相同签名的JavaScript函数：

```nim
# 2235. Add Two Integers
proc sum(num1, num2: int): int =
  num1 + num2
```

直接将上述代码编译为JavaScript的话会发现输出里没有这个函数。这是因为其没有被使用到。需要用`{.exportc.}`将其标记为导出[3]：

```nim
# 2235. Add Two Integers
proc sum(num1, num2: int): int {.exportc.} =
  num1 + num2
```

## 3. 原生数据类型

导出的函数需接收JavaScript中不同类型的参数。Nim中预定义了与C类型对应的兼容类型如`cint`、`cdouble`等，但在JavaScript中只会用到其中有限的几种。

### 3.1 `Boolean`

Nim中用`bool`接收。

### 3.2 `Null`

Nim中用`nil`接收。

### 3.3 `Undefined`

Nim中没有对应的类型，无法对其进行操作，但可以传递。

此外，在用于判断真值的时候可以用`nil`代替，因为Nim生成的JavaScript代码会用`==`进行比较，而在JavaScript中`undefined == null`。不清楚其中微妙后果的读者可以参考[这篇](/2021/04/24/jvm-ts.html)。

### 3.4 `Array`

Nim中可用`seq`接收。

### 3.5 `Number`

从`int`，`clonglong`到`cdouble`等都可以用来接收。由于JavaScript中的`Number`是64位浮点数，所以最准确的对应类型应该是`cdouble`/`float64`。但考虑到类型转换的开销以及LeetCode上题目的数据范围，大部分情况下用`int`就可以，对应32位整数。

显然，如果在Nim中使用了`int64`，则在JavaScript中可能会损失精度。在JavaScript中，需要用到64位整数的地方就只能用`BigInt`，并会带来一定的性能开销。

### 3.6 `BigInt`

Nim中需要用`JsBigInt`[4]。比如在对数字进行取模运算时，牵涉到乘法的时候：

```nim
import std/jsbigints

const MOD = 1e9.int + 7

type modInt = distinct int

proc `*`(x, y: modInt): modInt =
  (x.int.big * y.int.big mod MOD.big).toNumber.modInt
```

### 3.7 `String`

Nim中只能用`cstring`来接收，存在一定的类型转换开销。所以原则上是能不转就不转，通过`std/cstrutils`[5]来直接处理`cstring`。然而从经验上来看大部分情况下是不够用的，还是需要转换。

* `cstring`到`string`：`$str`
* `string`到`cstring`：`str.cstring`

## 4. 自定义类型

光能接收原生类型显然是不够的。像LeetCode上一些基础的面试题如翻转链表/二叉树等，就需要接收自定义的数据类型：

```nim
# 104. Maximum Depth of Binary Tree
type
  TreeNode = ref object
    val: int
    left, right: TreeNode

proc maxDepth(root: TreeNode): int {.exportc.} =
  if root == nil: 0
  else: 1 + max(maxDepth(root.left), maxDepth(root.right))
```

由于Nim中的`object`是值传递的，需在编译时确定大小，所以定义递归类型时必须使用引用（`ref`），否则编译器会报错。

此外，定义递归类型时通常会用到`{.acyclic.}`[6]，但由于编译成JavaScript以后就靠JavaScript进行垃圾回收了，所以在这里也就没必要使用了。

## 5. 使用闭包时的注意事项

在开启`-d:danger`选项时，如果用闭包捕获值类型的变量，运行时会发生错误`ReferenceError: nimCopy is not defined`：

```nim
proc median(arr: seq[int]): int {.exportc.} =
  proc medianInner(): int = arr[arr.len shr 1]
  result = medianInner()
```

解决办法是手动复制一下需捕获的变量：

```diff
proc median(arr: seq[int]): int {.exportc.} =
+  let arr = arr
  proc medianInner(): int = arr[arr.len shr 1]
  result = medianInner()
```

使用`-d:release`或`-d:debug`时则没有这个问题。

## 6. 导入函数

LeetCode上有一类交互题，要求你实现的函数去调用另一个黑箱的函数。这就需要在Nim中声明JavaScript中的函数：

```nim
# 374. Guess Number Higher or Lower
proc guess(num: int): int {.importc.}

proc guessNumber(n: int): int {.exportc.} =
  var (lo, hi) = (1, n + 1)
  while lo < hi:
    let mid = lo + ((hi - lo) shr 1)
    if guess(mid) > 0: lo = mid + 1
    else: hi = mid
  lo
```

JavaScript中的函数通过`{.importc.}`[7]声明就能在Nim中调用，Nim中的函数通过`{.exportc.}`声明就能在JavaScript中调用，相当方便。

Nim中的FFI还有更精细的控制，同样的功能也可以用其他的pragma实现[8]，比如：

```nim
proc guess(num: int): int {.importjs: "guess(#)".}
```

甚至是：

```nim
proc guess(num: int): int = {.emit: "return guess(`num`)".}
```

具体的使用场景下面会讲到。

## 7. 导入对象方法

LeetCode上的另一类交互题要求调用一个给定对象中的方法。还是一样的思路：

```nim
# 1237. Find Positive Integer Solution for a Given Equation
type CustomFunction = ref object of JsRoot
proc f(cf: Customfunction, x, y: int): int {.importcpp.}

const MAX = 1e3.int

proc findSolution(customFunction: CustomFunction, z: int): seq[seq[int]] {.exportc.} =
  var (x, y) = (1, MAX)
  while x <= MAX:
    while y >= 1 and customFunction.f(x, y) > z: y.dec
    if y >= 1 and customFunction.f(x, y) == z: result.add(@[x, y])
    x.inc
```

区别在于导入的对象方法需要用`{.importcpp.}`[9]而不是`{.importc.}`。有兴趣的读者可以比对下这两者编译出来的JavaScript代码。

## 8. 导出对象

LeetCode上还有一类设计题，要求实现一个类，驱动的JavaScript代码会构造这个类的对象并调用其上的方法。比如[1603. Design Parking System](https://leetcode.cn/problems/design-parking-system/)，其代码模版是这样的：

```javascript
/**
 * @param {number} big
 * @param {number} medium
 * @param {number} small
 */
var ParkingSystem = function(big, medium, small) {

};

/** 
 * @param {number} carType
 * @return {boolean}
 */
ParkingSystem.prototype.addCar = function(carType) {

};

/**
 * Your ParkingSystem object will be instantiated and called as such:
 * var obj = new ParkingSystem(big, medium, small)
 * var param_1 = obj.addCar(carType)
 */
```

这个问题可分成两步解决。

### 8.1 导出构造函数

这一步的关键是，JavaScript中的函数在被当作类的构造函数（通过`new`调用）时，`this`对象会指向类的新实例，并会在函数返回时被隐式返回[10]。由于Nim中只有普通的函数，就需要重现这个逻辑：

```nim
type ParkingSystem = ref object
  slots: seq[int]

proc newParkingSystem(big: int, medium: int, small: int): ParkingSystem {.exportc: "ParkingSystem".} =
  var this {.importc, nodecl.}: ParkingSystem

  this.slots = @[0, big, medium, small]

  this
```

在这里，由于`this`是隐式声明的，所以需要用`{.nodecl.}`[11]来跳过变量声明的生成。还可以看到`{.exportc.}`可以支持别名，所以Nim中的函数名可以与JavaScript中的对应函数名不一致。

### 8.2 导出方法

这一步的关键是，需要把方法绑定到类对象的`prototype`上。为此这里用上了`{.emit.}`[12]：

```nim
proc addCar(this: ParkingSystem, carType: int): bool =
  if this.slots[carType] == 0: false
  else: this.slots[carType].dec; true

{.emit: "ParkingSystem.prototype.addCar = addCar".}
proc addCarJs(carType: int): bool {.exportc: "addCar".} =
  var this {.importc, nodecl.}: ParkingSystem
  this.addCar(carType)
```

`{.emit.}`可以直接输出目标代码，非常强大也非常容易被滥用，因为如果全部代码都用其来生成的话，还不如直接用目标语言来写呢。

此外这里多声明了一个wrapper函数，在其中导入了隐式的`this`对象，作为对应的Nim函数的第一个参数传递。这样Nim对象的方法之间也能够相互调用；去掉wrapper函数就是纯粹的Nim代码，语言之间的分界线也变得更加明显。

由于声明wrapper需要写很多脚手架代码，我已在之前的LeetCode解题工具中[13]加入了自动生成脚手架的模版。

## 9. 小结

至此，在LeetCode出现的绝大部分题目都可以用Nim来解决了。下一回我们会讲讲Nim标准库和JavaScript标准库在互操作中的问题。

## 参考资料

* [1] [Nim Compiler User Guide - Compiler Usage - Command-line switches](https://nim-lang.org/docs/nimc.html#compiler-usage-commandminusline-switches)
* [2] [Nim Compiler User Guide - Additional compilation switches](https://nim-lang.org/docs/nimc.html#additional-compilation-switches)
* [3] [Nim Manual - Foreign function interface - Exportc pragma](https://nim-lang.org/docs/manual.html#foreign-function-interface-exportc-pragma])
* [4] [std/jsbigints](https://nim-lang.org/docs/jsbigints.html)
* [5] [std/cstrutils](https://nim-lang.org/docs/cstrutils.html)
* [6] [Nim Manual - Pragmas - acyclic pragma](https://nim-lang.org/docs/manual.html#pragmas-acyclic-pragma)
* [7] [Nim Manual - Foreign function interface - Importc pragma](https://nim-lang.org/docs/manual.html#foreign-function-interface-importc-pragma)
* [8] [Nim for TypeScript Programmers - JavaScript interoperability](https://github.com/nim-lang/Nim/wiki/Nim-for-TypeScript-Programmers#javascript-interoperability)
* [9] [Nim Manual - Implementation Sepcific Pragmas - ImportCpp pragma](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-importcpp-pragma)
* [10] [new operator - JavaScript \| MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)
* [11] [Nim Manual - Implementation Sepcific Pragmas - nodecl pragma](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-nodecl-pragma)
* [12] [Nim Manual - Implementation Sepcific Pragmas - Emit pragma](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-emit-pragma)
* [13] [ts-leetcode/class.hbs at nim - qszhu/ts-leetcode](https://github.com/qszhu/ts-leetcode/blob/nim/templates/class.hbs)
