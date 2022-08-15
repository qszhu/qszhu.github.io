---
layout: post
title: 在编程竞赛中使用Kotlin（一）
date: 2022-08-15 18:00:00 +0800
tags: [kotlin, competitive_programming]
---

# 目录
* 1 背景
* 2 理想的语言
* 3 Kotlin
* 4 环境配置
* 5 实战
  * 5.1 基础语法
    * 5.1.1 differentSymbolsNaive
    * 5.1.2 Knapsack Light
    * 5.1.3 isIPv4Address
  * 5.2 函数式编程与Lambda
    * 5.2.1 extractEachKth
    * 5.2.2 alternatingSums
  * 5.3 Range和Sequence
    * 5.3.1 avoidObstacles
    * 5.3.2 commonCharacterCount
    * 5.3.3 matrixElementSum
  * 5.4 作用域函数
    * 5.4.1 Array Replace
    * 5.4.2 Add Border
  * 5.5 频次统计
    * 5.5.1 palindromeRearranging
  * 5.6 （滑动）窗口
    * 5.6.1 adjacentElementsProduct
    * 5.6.2 almostIncreasingSequence
    * 5.6.3 messageFromBinaryCode
  * 5.7 前缀和
    * 5.7.1 arrayMaxConsecutiveSum
* 6 小结
* 7 参考资料

## 1. 背景

[之前](/2020/08/17/leetcode-with-typescript.html)提到，由于机缘巧合开始刷LeetCode[1]。后来逐渐不满足于只刷LeetCode，还开始刷CodeForces[2]和AtCoder[3]等。不过这时JavaScript/TypeScript作为脚本语言在编程竞赛中的劣势逐渐体现，主要表现在数据规模大时容易被卡常（指实现相同复杂度的算法，由于运行时的额外开销导致运行超时），让我把目光转向其他替代语言。

## 2. 理想的语言

我理想中的编程竞赛语言要能满足几个条件：

### 开发速度快

这点是脚本语言的优势。因为编程竞赛中总会有仅需直接实现的简单题（俗称签到题），能尽快完成这类题目意味着有更多时间可以花在更难的题目上。在这一点上因为众所周知的原因，Rust对新手就不是很适合。

### 表达能力强

这点是多范式语言的优势。有的题目比如构造题，要求能够尽快把想到的构造方法实现出来，这时语言就不能因为某些特定的要求而妨碍你把想法表达出来。有的题可能用过程式的代码实现起来方便，而有的题用函数式实现起来方便，那么单纯的函数式语言就不是很适合，比如Haskell。Go可能也不是很适合，否则社区根本就不会为了加入泛型支持吵那么久。

### 运行效率高

这点是脚本语言的弱项。JavaScript尽管有V8这样高度优化的引擎，运行效率依旧被语言特性所拖累，才会有asm.js和wasm等相继被提出。而Python在PyPy的加持下似乎尚可一战。

### 官方支持

如果标程没有用你所使用的语言写过，那你就会有更大的概率遇到出题人没有考虑到情况。而竞赛的唯结果论意味着要跟出题人对拍子而不是给人找bug。这就连同自制语言一起，把各种小众语言一并排除了。对于ACM竞赛来讲，语言就被限定在了C/C++、Java和Python，以及……Kotlin？

## 3. Kotlin

由于退役太久，我之前并知道ACM总决赛都用上Kotlin了[4]。而在我不做Android开发好多年后，才听说似乎有用Kotlin开发的Android应用。大致了解了一下之后，发现Kotlin基本能满足上面提到的条件：

* 开发速度快：Kotlin相比Java有更精简的语法，并且能够以脚本的方式运行；
* 表达能力强：Kotlin支持多种编程范式，过程式、函数式、面向对象和泛型等；
* 运行效率高：由于运行在JVM上，Java能过的题Kotlin应该也都能过；
* 官方支持：这点上面已经提到过了。在Kotlin的文档中，甚至专门有一篇讲如何在编程竞赛中使用Kotlin的简要文章[5]。

于是我决定通过实战再深入了解一下。

## 4. 环境配置

实战之前先配置环境。试下来第一方的IDE对于Kotlin的支持是最好的，也就是IntelliJ IDEA。社区版就够用，新建个工程就可以开始写代码了。对于编程竞赛来说，我习惯复用同一个工程，毕竟只是需要一些语法检查和自动补全功能而已。

## 5. 实战

在尝试一门新语言时，我通常会去Codewars[6]，Exercism[7]或CodeSignal(原CodeFights)[8]上做些简单的题熟悉一下。这里我们来看些CodeSignal上的题。

### 5.1 基础语法

#### 5.1.1 [differentSymbolsNaive](https://app.codesignal.com/arcade/intro/level-8/8N7p3MqzGQg5vFJfZ)

##### 题目描述

求字符串中不同字符的个数。例如当`s = "cabca"`时，`solution(s) = 3`。

##### 解答

直观的做法是把字符串中的字符都加入一个哈希表，最后返回哈希表的大小。例如：

```kotlin
fun solution(s: String): Int {
    val chars = HashSet<Char>()
    for (c in s.toCharArray()) {
        chars.add(c)
    }
    return chars.size
}
```

从上面这段代码可以看到，Kotlin可以无缝使用Java的API，比如`HashSet`，`.toCharArray()`等。除了声明函数和变量等的语法略有不同以外，跟普通的Java程序差别不大。熟悉其他语言的用户也很容易理解。

不过使用Kotlin能把代码写得更简洁：

```kotlin
fun solution(s: String): Int = s.toSet().size
```

从上面这段代码可以看到，Kotlin中进行类型转换需要通过`.toXXX()`进行，比如`.toString()`、`.toInt()`等，甚至`String`通过`.toSet()`就直接成了`Set<Char>`。

另外，如果函数体只有一个表达式的话，可以直接用`=`连接函数声明，省略大括号和`return`。

#### 5.1.2 [Knapsack Light](https://app.codesignal.com/arcade/intro/level-9/r9azLYp2BDZPyzaG2)

##### 题目描述

两个物品，重量分别为`weight1`和`weight2`，价值分别为`value1`和`value2`。现有容量为`maxW`的背包，问能带走的最大物品价值为多少。

##### 解答

两个物品的0-1背包问题，可以直接枚举。比如：

```kotlin
fun solution(value1: Int, weight1: Int, value2: Int, weight2: Int, maxW: Int): Int {
    if (weight1 > maxW && weight2 > maxW) return 0
    if (weight1 > maxW) return value2
    if (weight2 > maxW) return value1
    if (weight1 + weight2 <= maxW) return value1 + value2
    return maxOf(value1, value2)
}
```

上面的代码中，最后一行用Java的`Math.max()`也是可以的，不过IDEA会提示说用Kotlin函数`coerceAtLeast()`代替。个人以为`coerceXXX()`函数实现的功能是clamp，在这里使用的话形式上不够直观，所以用了另一个在形式上更接近的函数`maxOf()`。

另外，像是这种不同条件的枚举在现代语言中都可以用match/when等语句替代，在Kotlin中也一样：

```kotlin
fun solution(value1: Int, weight1: Int, value2: Int, weight2: Int, maxW: Int): Int =
    when {
        weight1 > maxW && weight2 > maxW  -> 0
        weight1 > maxW                    -> value2 
        weight2 > maxW                    -> value1
        weight1 + weight2 <= maxW         -> value1 + value2
        else                              -> maxOf(value1, value2)
    }
```

#### 5.1.3 [isIPv4Address](https://app.codesignal.com/arcade/intro/level-5/veW5xJednTy4qcjso)

##### 题目描述

判断一个字符串是否是合法的IPv4地址。如`inputString = "172.16.254.1"`时，`solution(inputString) = true`。

##### 解答

一种直观的方法是，把字符串用`"."`分割后判断每一部分是否符合条件：

```kotlin
fun solution(inputString: String): Boolean {
    val parts = inputString.split(".")
    return parts.size == 4 && parts.all { it.toIntOrNull() in 0..255 && (it == "0" || !it.startsWith("0")) }
}
```

这里我们可以看到：

* 因为每一部分的字符串可能不是合法的整数，所以转换时用了`.toIntOrNull()`而不是`.toInt()`，这样在转换失败时会返回`null`而不是抛出异常。Kotlin的API中以`OrNull`结尾的方法都是这样的逻辑；
* 在判断整数是否在范围内时用了`in Range`的形式，比写`if`更简洁，并且不需要额外再判断`null`；
* 这里用到了lambda，熟悉函数式编程的同学理解起来应该没啥困难。留到下一题再详细说说。

此外，因为函数体有多条语句，所以没有用`=`的形式。那么能否用`=`呢？也是可以的：

```kotlin
fun solution(inputString: String): Boolean =
    inputString.let {
        val parts = it.split(".")
        parts.size == 4 && parts.all { part ->
            part.toIntOrNull() in 0..255 && (part == "0" || !part.startsWith("0"))
        }
    }
```

这里的`let`创建了一个新的作用域，而本身可以作为一个表达式返回，所以可以用`=`。在这里使用的话，除了形式上的区别，并没有带来什么实质上的好处（可能只是让熟悉ML系语言的同学觉得眼熟，包括像前面的`when`语句）。不过Kotlin中的`let`是所谓的**作用域函数**中的一个，后面我们还会看到其他的作用域函数。

这题还可以用正则表达式做：

```kotlin
fun solution(inputString: String): Boolean =
    inputString.let {
        val num = """([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])"""
        """^($num\.){3}$num$""".toRegex().matches(it)
    }
```

这里我们可以看到：

* 在Kotlin中要构造一个正则表达式，可以用`String.toRegex()`，也可以直接用构造函数`Regex()`；
* 用三引号`"""`括起来的字符串就是raw string，可以避免Java中使用正则表达式时需要二次转义的问题；
* 可以用`${}`在字符串中插入表达式的值，表达式只有变量名的时候还可以省略掉`{}`。因为插入是在字符串转换为正则表达式之前进行的，所以`${}`并不会和正则表达式中的`$`运算符或`{}`控制字符相混淆。

下面我们来看看Kotlin中的函数式编程。

### 5.2 函数式编程与Lambda

#### 5.2.1 [extractEachKth](https://app.codesignal.com/arcade/intro/level-8/3AgqcKrxbwFhd3Z3R)

##### 题目描述

删除一个数组中每第k个数字。如`inputArray = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]`，`k = 3`，则`solution(inputArray, k) = [1, 2, 4, 5, 7, 8, 10]`

##### 解答

根据数组下标来筛选元素，Kotlin中通过`.filterIndexed()`方法来实现：

```kotlin
fun solution(inputArray: List<Int>, k: Int): List<Int> =
    inputArray.filterIndexed({ index, i -> (index + 1) % k != 0 })
```

如果这样写的话，IDEA会提示把lambda拿到括号外。因为在Kotlin中，如果lambda是函数的最后一个参数，则可以把lambda拿出参数列表；如果lambda是唯一的参数，则参数列表的括号也可以省略：

```kotlin
fun solution(inputArray: List<Int>, k: Int): List<Int> =
    inputArray.filterIndexed { index, i -> (index + 1) % k != 0 }
```

这时IDEA还会提示说lambda中的第二个参数没有用到，可以用`_`替代：

```kotlin
fun solution(inputArray: List<Int>, k: Int): List<Int> =
    inputArray.filterIndexed { index, _ -> (index + 1) % k != 0 }
```

此外，`Int`类型上还有`.rem()`和`.mod()`等扩展方法可以实现取模的功能，但用在这里和`%`运算符没有实质的区别。不如在扩展方法中用一下：

```kotlin
fun Int.divides(n: Int): Boolean = this.rem(n) == 0

fun solution(inputArray: List<Int>, k: Int): List<Int> =
    inputArray.filterIndexed { index, _ -> !(index + 1).divides(k) }
```

可以看到，在Kotlin中可以很方便地为类型扩展新的方法，这就使得Kotlin的表达能力大大增强了。

#### 5.2.2 [alternatingSums](https://app.codesignal.com/arcade/intro/level-4/cC5QuL9fqvZjXJsW9)

##### 题目描述

将数组中的元素按奇偶位置分别求和。例如`a = [50, 60, 60, 45, 70]`时，`solution(a) = [180, 105]`。

##### 解答

把数组按照下标分成两部分再分别求和。用`.withIndex()`和`.partition()`就可以写得很流畅：

```kotlin
fun solution(a: List<Int>): List<Int> =
    a.withIndex()
        .partition { vi -> vi.index.divides(2) }.toList()
        .map { it.sumOf { vi -> vi.value } }
```

可以看到代码基本就是对自然语言算法的翻译，顺便还用到了上面自定义的扩展方法。

### 5.3 Range和Sequence

上面曾短暂地接触了一下`Range`，这里再来看几道题。

#### 5.3.1 [avoidObstacles](https://app.codesignal.com/arcade/intro/level-5/XC9Q2DhRRKQrfLhb5)

##### 题目描述

找到最小的不能被数组中所有元素整除的整数。如`inputArray = [5, 3, 6, 7, 9]`时，`solution(inputArray) = 4`。

##### 解答

直接翻译即可：

```kotlin
fun solution(inputArray: MutableList<Int>): Int =
    (1..1001).first { d -> inputArray.all { !it.divides(d) } }
```

这里根据题目的数据范围用了固定的`Range`。如果觉得不够通用的话，还可以通过`generateSequence()`来生成无限流：

```kotlin
fun solution(inputArray: MutableList<Int>): Int =
    generateSequence(1) { it + 1 }.first { d -> inputArray.all { !it.divides(d) } }
```

#### 5.3.2 [commonCharacterCount](https://app.codesignal.com/arcade/intro/level-3/JKKuHJknZNj4YGL32)

##### 题目描述

找到两个字符串中相同字母的数量。如`s1 = "aabcc"`，`s2 = "adcaa"`时，`solution(s1, s2) = 3`。

##### 解答

直接翻译即可：

```kotlin
fun solution(s1: String, s2: String): Int =
    ('a'..'z').sumOf { c ->
        minOf(s1.count { it == c }, s2.count { it == c })
    }
```

可以看到除了整数能构造`Range`外，字符也可以。

#### 5.3.3 [matrixElementsSum](https://app.codesignal.com/arcade/intro/level-2/xskq4ZxLyqQMCLshr)

##### 题目描述

找出一个矩阵中每一列上第一个0上面的数字之和。如
```
matrix = [[0, 1, 1, 2], 
          [0, 5, 0, 0], 
          [2, 0, 3, 3]]
```
时，`solution(matrix) = 9`。

##### 解答

直接翻译即可：

```kotlin
fun solution(matrix: MutableList<MutableList<Int>>): Int =
    matrix[0].indices.sumOf { c ->
        matrix.indices.map { r -> matrix[r][c] }
            .takeWhile { it > 0 }
            .sum()
    }
```

其中`.indices`属性给出了一种更简洁的遍历数组下标的方法。

再来看看作用域函数。

### 5.4 作用域函数

#### 5.4.1 [Array Replace](https://app.codesignal.com/arcade/intro/level-6/mCkmbxdMsMTjBc3Bm)

##### 题目描述

将数组中所有的值为a的元素替换为b。如`inputArray = [1, 2, 1]`，`elemToReplace = 1`，`substitutionElem = 3`时，`solution(inputArray, elemToReplace, substitutionElem) = [3, 2, 3]`。

##### 解答

用`.map()`的话很容易实现：

```kotlin
fun solution(inputArray: MutableList<Int>, elemToReplace: Int, substitutionElem: Int): List<Int> =
    inputArray.map { if (it == elemToReplace) substitutionElem else it }
```

注意到Kotlin中的`if`是表达式，所以可以直接用在lambda里，就不需要像某些语言一样用三元操作符`?:`。

在Kotlin的API中可以发现`.replaceAll()`方法，不过没有返回值，难道只能写成多行的函数体再显式`return`吗？这里可以利用Kotlin的作用域函数`.apply()`：

```kotlin
fun solution(inputArray: MutableList<Int>, elemToReplace: Int, substitutionElem: Int): List<Int> =
    inputArray.apply { inputArray.replaceAll { if (it == elemToReplace) substitutionElem else it } }
```

对于这种没有返回值的方法，`.apply()`就可以在调用完方法后返回被调用者。

#### 5.4.2 [Add Border](https://app.codesignal.com/arcade/intro/level-4/ZCD7NQnED724bJtjN)

##### 题目描述

给一组字符串加个边框。如
```
picture = ["abc",
           "ded"]
```
时，
```
solution(picture) = ["*****",
                     "*abc*",
                     "*ded*",
                     "*****"]
```

##### 解答

这里可以对函数入参作一系列的操作再返回，用`.apply()`就很合适：

```kotlin
fun solution(picture: MutableList<String>): MutableList<String> =
    picture.apply {
        replaceAll { "*$it*" }
        val row = "*".repeat(this[0].length)
        add(row).also { add(0, row) }
    }
```

注意到这里还用了另一个作用域函数`.also()`，不过在这里的作用就仅限于把两个语句写在一行里。`.also()`也和`.apply()`一样返回被调用者。在Kotlin里用来交换两个变量值的惯用写法就是用`.also()`实现的：

```kotlin
a = b.also { b = a }
```

虽然用`.apply()`也可以达到同样的效果，但在上述场合下`.also()`表达的意思更清晰。

最后来看几个竞赛中比较常见的需求的实现。

### 5.5 频次统计

#### 5.5.1 [palindromeRearranging](https://app.codesignal.com/arcade/intro/level-4/Xfeo7r9SBSpo3Wico)

##### 题目描述

判断一个字符串重新排列后能否构成回文串。如`inputString = "aabb"`时，`solution(inputString) = true`。

##### 解答

统计字符串中各个字符出现的频次，要求至多只能有一个出现奇数次的字符。统计频次可以用`.groupBy()`：

```kotlin
fun solution(inputString: String): Boolean =
    inputString.groupBy { it }.values.count { !it.size.divides(2) } <= 1
```

不过由于我们只关心字符出现的频次而不关心具体的字符，这里可以用`.groupingBy()`和`.eachCount()`，性能会更好：

```kotlin
fun solution(inputString: String): Boolean =
    inputString.groupingBy { it }.eachCount().values.sumOf { it % 2 } <= 1
```

### 5.6 （滑动）窗口

#### 5.6.1 [adjacentElementsProduct](https://app.codesignal.com/arcade/intro/level-2/xzKiBHjhoinnpdh6m)

##### 题目描述

求数组中相邻两数的乘积的最大值。如`inputArray = [3, 6, -2, -5, 7, 3]`时，`solution(inputArray) = 21`。

##### 解答

Kotlin中有专门的方法用来遍历数组中相邻的两个数：

```kotlin
fun solution(inputArray: MutableList<Int>): Int =
    inputArray.zipWithNext { a, b -> a * b }.maxOf { it }
```

在函数式编程中有一种风格叫做“隐式编程”(Point-free)，即在把函数作为参数传递时，不显式写出该函数的参数。在Kotlin中可以通过**函数引用**来实现：

```kotlin
fun solution(inputArray: MutableList<Int>): Int =
    inputArray.zipWithNext(Int::times).maxOf { it }
```

#### 5.6.2 [almostIncreasingSequence](https://app.codesignal.com/arcade/intro/level-2/2mxbGwLzvkTCKAJMG)

##### 题目描述

最多允许从数组中移除一个元素，问数组中的元素是否是升序排列的。如`sequence = [1, 3, 2]`时，`solution(sequence) = true`。

##### 解答

分别统计相邻和间隔一个元素的逆序数对个数。相邻元素可以用`.zipWithNext()`，间隔一个元素的话则需要用上`.windowed()`：

```kotlin
fun solution(sequence: MutableList<Int>): Boolean =
    sequence.zipWithNext { a, b -> a >= b }.count { it } <= 1 &&
    sequence.windowed(3).count { it[0] >= it[2] } <= 1
```

可以看到`.zipWithNext()`其实就相当于`.windowed(2)`。而`.windowed()`则有更多的参数，可以控制窗口的大小和移动的步长等，相比`.zipWithNext()`更通用。

当然也可以用`Range`来遍历数组的下标：

```kotlin
fun solution(sequence: MutableList<Int>): Boolean =
    (1 until sequence.size).count { sequence[it - 1] >= sequence[it] } <= 1 &&
    (2 until sequence.size).count { sequence[it - 2] >= sequence[it] } <= 1
```

Kotlin中闭区间用`..`构建，半开区间用`until`，降序则需要用`downTo`。IDEA中对于`Range`有很直观的提示，不用担心会搞错。

#### 5.6.3 [messageFromBinaryCode](https://app.codesignal.com/arcade/intro/level-12/sCpwzJCyBy2tDSxKW)

##### 题目描述

二进制串转为字符串。如`code = "010010000110010101101100011011000110111100100001"`时，`solution(code) = "Hello!"`。

##### 解答

将字符串切成8个字符一组，每组先转换为整数再转换为字符，最后拼接。切割可以通过`.windowed()`来实现：

```kotlin
fun solution(code: String): String =
    code.windowed(8, 8).map { it.toInt(2).toChar() }.joinToString("")
```

不过因为这里窗口之间没有重叠，用`.chunked()`会更合适：

```kotlin
fun solution(code: String): String =
    code.chunked(8) { it.toString().toInt(2).toChar() }.joinToString("")
```

而如果意识到这个二进制串其实代表了一个`Byte`数组，也可以从`Byte`数组来构造字符串：

```kotlin
fun solution(code: String): String =
    code.chunked(8) { it.toString().toByte(2) }.toByteArray().decodeToString()
```

### 5.7 前缀和

#### 5.7.1 [arrayMaxConsecutiveSum](https://app.codesignal.com/arcade/intro/level-8/Rqvw3daffNE7sT7d5)

##### 题目描述

求数组中长度为k的子数组中的最大子数组和。如`inputArray = [2, 3, 5, 1, 6]`， `k = 2`，则`solution(inputArray, k) = 8`。

##### 解答

最最基本的滑动窗口。如果用`.windowed()`可以直接翻译出来：

```kotlin
// TLE
fun solution(inputArray: MutableList<Int>, k: Int): Int =
    inputArray.windowed(k).maxOf { it.sum() }
```

不过这个窗口并没有真正“滑动”起来，会导致很多重复计算（时间复杂度`O(nk)`），数据规模大的时候就可能会超时。

Kotlin并没有强制要求用某种风格写代码，所以可以用传统的过程式风格写（时间复杂度`O(n)`）：

```kotlin
fun solution(inputArray: MutableList<Int>, k: Int): Int {
    var sum = inputArray.slice(0 until k).sum()
    var res = sum
    for (i in k until inputArray.size) {
        sum += inputArray[i] - inputArray[i - k]
        res = res.coerceAtLeast(sum)
    }
    return res
}
```

发现上面代码中循环里做的事情可以用accumulate/reduce实现。不过Kotlin中的`.reduce()`无法指定初始值，需要用`.fold()`：

```kotlin
fun solution(inputArray: MutableList<Int>, k: Int): Int =
    inputArray.let { a ->
        var sum = a.slice(0 until k).sum()
        (k until a.size).fold(sum) { acc, i ->
            acc.let {
                sum += a[i] - a[i - k]
                it.coerceAtLeast(sum)
            }
        }
    }
```

不过个人感觉这样的代码可读性反而变差了。

于是我们换种思路。因为需要求和，所以可以用前缀和来做。一样是fold，不过需要把每次fold的结果都保留下来，Kotlin中可以用`.runningFold()`来实现：

```kotlin
fun solution(inputArray: MutableList<Int>, k: Int): Int =
    inputArray.let { a ->
        val preSum = a.runningFold(0) { acc, i -> acc + i }
        (k..a.size).maxOf { i -> preSum[i] - preSum[i - k] }
    }
```

而`.runningFold()`还有另一个名字叫`.scan()`，可以少打很多字母：

```kotlin
fun solution(inputArray: MutableList<Int>, k: Int): Int =
    inputArray.let { a ->
        val preSum = a.scan(0) { acc, i -> acc + i }
        (k..a.size).maxOf { i -> preSum[i] - preSum[i - k] }
    }
```

## 6. 小结

经过这些简单的题目试下来，看上去Kotlin的开发速度可以媲美脚本，足以应付签到题。还可以看到Kotlin支持不同的代码风格，每道题可以有多种不同的实现方式（不像Python有那种所谓的“只有一种实现方式”的哲学），就编程竞赛来讲是足够胜任的。下次我们可以看看Kotlin对于更复杂的题目会有什么样的表现。

## 7. 参考资料

* [1] [LeetCode - The World’s Leading Online Programming Learning Platform](https://leetcode.com/)
* [2] [Codeforces](https://codeforces.com/)
* [3] [AtCoder](https://atcoder.jp/)
* [4] [JetBrains to support the ACM-ICPC \| JetBrains News](https://blog.jetbrains.com/blog/2017/05/23/jetbrains-to-support-the-acm-icpc/)
* [5] [Kotlin for competitive programming \| Kotlin](https://kotlinlang.org/docs/competitive-programming.html#learning-kotlin)
* [6] [Codewars - Achieve mastery through coding practice and developer mentorship](https://codewars.com/)
* [7] [Exercism](https://exercism.org/)
* [8] [CodeSignal](https://app.codesignal.com/)