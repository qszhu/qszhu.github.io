---
layout: post
title: 用TypeScript实现一门语言(3)——语法分析拾遗
date: 2021-09-07 21:00:00 +0800
tags: [parsing, leetcode]
---

### 目录
* [用TypeScript实现一门语言(1)——语法分析](/2021/08/22/parser-combinators.html)
* [用TypeScript实现一门语言(2)——表达式解析](/2021/08/27/parsing-expressions.html)
* 用TypeScript实现一门语言(3)——语法分析拾遗 <-- 你在这里

俗话说，手里拿着锤子看啥都像钉子。我们现在有了个强大的解析工具，就让我们来尝试解决一些问题，权当练习。

我从LeetCode上选出了一些能用语法分析技术解决的问题：

* [[困难] 224. 基本计算器](https://leetcode-cn.com/problems/basic-calculator/)
* [[中等] 227. 基本计算器 II](https://leetcode-cn.com/problems/basic-calculator-ii/)
* [[中等] 385. 迷你语法分析器](https://leetcode-cn.com/problems/mini-parser/)
* [[中等] 394. 字符串解码](https://leetcode-cn.com/problems/decode-string/)
* [[中等] 439. 三元表达式解析器](https://leetcode-cn.com/problems/ternary-expression-parser/)
* [[困难] 726. 原子的数量](https://leetcode-cn.com/problems/number-of-atoms/)
* [[困难] 736. Lisp语法解析](https://leetcode-cn.com/problems/parse-lisp-expression/)
* [[困难] 770. 基本计算器 IV](https://leetcode-cn.com/problems/basic-calculator-iv/)
* [[困难] 772. 基本计算器 III](https://leetcode-cn.com/problems/basic-calculator-iii/)
* [[困难] 1096. 花括号展开 II](https://leetcode-cn.com/problems/brace-expansion-ii/)
* [[困难] 1106. 解析布尔表达式](https://leetcode-cn.com/problems/parsing-a-boolean-expression/)

让我们一题一题来看。

(顺带一提，我是用我之前写的[LeetCode解题工具](/2020/08/16/leetcode-with-typescript.html)来提交TypeScript的代码，这样就可以把parser combinators作为一个库文件import进来)

### [[中等] 394. 字符串解码](https://leetcode-cn.com/problems/decode-string/)

这题的输入是这样的：

```
Example 1:

Input: s = "3[a]2[bc]"
Output: "aaabcbc"

Example 2:

Input: s = "3[a2[c]]"
Output: "accaccacc"

Example 3:

Input: s = "2[abc]3[cd]ef"
Output: "abcabccdcdcdef"

Example 4:

Input: s = "abc3[cd]xyz"
Output: "abccdcdcdxyz"

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/decode-string
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

语法规则可以写成一条：

```
encoded -> (Str | Num "[" encoded "]")*
```

对应的实现是这样的：

```typescript
// encoded -> (Str | Num "[" encoded "]")*
const encoded = lazy(() =>
  zeroOrMore(
    oneOf(
      Str,
      seqOf(Num, str('['), encoded, str(']'))
    )
  ))
```

直接解析出来则是这样的：

```typescript
3[a]2[bc]
[ [ 3, '[', [ 'a' ], ']' ], [ 2, '[', [ 'bc' ], ']' ] ]
3[a2[c]]
[ [ 3, '[', [ 'a', [ 2, '[', [ 'c' ], ']' ] ], ']' ] ]
2[abc]3[cd]ef
[ [ 2, '[', [ 'abc' ], ']' ], [ 3, '[', [ 'cd' ], ']' ], 'ef' ]
abc3[cd]xyz
[ 'abc', [ 3, '[', [ 'cd' ], ']' ], 'xyz' ]
```

虽然也不是不能用，但匹配到的中括号`[`和`]`其实是不需要的。让我们尝试用`.map()`把这些中括号去掉：

```typescript
// encoded -> (Str | Num "[" encoded "]")*
const encoded: Parser = lazy(() =>
  zeroOrMore(oneOf(Str, seqOf(Num, str('['), encoded, str(']')))))
    .map(res => res.map((r: any) => Array.isArray(r) ? [r[0], r[2]] : r))
```

乍一看可能有些绕。让我们来分解一下。首先`zeroOrMore`返回一个数组，所以传入`.map()`的参数是一个数组。数组中的元素有两种可能(对应`oneOf`)：一种是字符串(对应`Str`)，另一种是数组(对应`seqOf`)。我们需要把一个数组中的数组元素里的方括号丢弃，于是就写成了上面的样子。这样确实可以达到我们想要的效果：

```typescript
3[a]2[bc]
[ [ 3, [ 'a' ] ], [ 2, [ 'bc' ] ] ]
3[a2[c]]
[ [ 3, [ 'a', [ 2, [ 'c' ] ] ] ] ]
2[abc]3[cd]ef
[ [ 2, [ 'abc' ] ], [ 3, [ 'cd' ] ], 'ef' ]
abc3[cd]xyz
[ 'abc', [ 3, [ 'cd' ] ], 'xyz' ]
```

只不过代码比较难看。由于这是种比较常见的模式，比如程序设计语言中数组用`[]`，函数参数列表用`()`，代码块用`{}`等，我们可以把这种模式抽象成parser构造函数`between`：

```typescript
const between = (left: Parser, content: Parser, right: Parser) =>
  seqOf(left, content, right)
    .map(([_, content]) => content)
```

这样构建的Parser，在有匹配的时候，就会把左右两边的匹配丢弃，只返回中间的匹配结果。于是上面的规则就可以这样写：

```diff
const encoded = lazy(() =>
-   zeroOrMore(oneOf(Str, seqOf(Num, str('['), encoded, str(']')))))
+   zeroOrMore(oneOf(Str, seqOf(Num, between(str('['), encoded, str(']'))))))
-     .map(res => res.map((r: any) => Array.isArray(r) ? [r[0], r[2]] : r))
```

你可以验证下这跟上面的写法是等价的，并且没有难看的`.map()`了。

更进一步，我们可以把`between` curry化：

```diff
- const between = (left: Parser, content: Parser, right: Parser) =>
+ const between = (left: Parser, right: Parser) => (content: Parser) =>
  seqOf(left, content, right)
    .map(([_, content]) => content)
```

上面规则的写法就可以进一步简化：

```diff
+ const betweenBrackets = between(str('['), str(']'))
+
const encoded = lazy(() =>
-   zeroOrMore(oneOf(Str, seqOf(Num, between(str('['), encoded, str(']'))))))
+   zeroOrMore(oneOf(Str, seqOf(Num, betweenBrackets(encoded)))))
```

`()`和`{}`也是一样的写法，甚至三元表达式的`:?`也可以这样写，后面我们会遇到。

### [[中等] 385. 迷你语法分析器](https://leetcode-cn.com/problems/mini-parser/)

这题的输入是这样的：

```
Example 1:

Input: s = "324"
Output: 324
Explanation: You should return a NestedInteger object which contains a single integer 324.

Example 2:

Input: s = "[123,[456,[789]]]"
Output: [123,[456,[789]]]
Explanation: Return a NestedInteger object containing a nested list with 2 elements:
1. An integer containing value 123.
2. A nested list containing two elements:
    i.  An integer containing value 456.
    ii. A nested list with one element:
         a. An integer containing value 789

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/mini-parser
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

语法规则可以这样写：

```
intList -> "[" nestedInt ("," nestedInt)* "]"
nestedInt -> Int | intList
```

这里的方括号`[]`我们就可以用上面的`between`来实现了。此外这里还出现了一个常见的模式就是以`,`分隔的列表。通常我们需要在`.map()`里把匹配到的`,`丢弃。我们也可以像上面一样再定义一个parser构造函数`sepBy1`：

```typescript
const sepBy1 = (sep: Parser) => (value: Parser) =>
  seqOf(value, zeroOrMore(seqOf(sep, value)))
    .map(([first, rest]) =>
      [first, ...(rest.map((r: any) => r[1]))])
```

规则就可以这样实现：

```typescript
const betweenBracket = between(str('['), str(']'))

const sepByComma = sepBy1(str(','))

// intList -> "[" nestedInt ("," nestedInt)* "]"
const intList = lazy(() =>
  betweenBracket(sepByComma(nestedInt)))

// nestedInt = Int | intList
const nestedInt = lazy(() =>
  oneOf(Int, intList))
```

不过这样是过不了测试用例的，因为测试用例里有空列表`"[]"`。所以其实语法规则是这样的：

```diff
- intList -> "[" nestedInt ("," nestedInt)* "]"
+ intList -> "[" (nestedInt ("," nestedInt)*)? "]"
nestedInt -> Int | intList
```

你可以试着自己改一下`sepBy1`的实现。

### [[困难] 1106. 解析布尔表达式](https://leetcode-cn.com/problems/parsing-a-boolean-expression/)

这题的输入是这样的：

```
Example 1:

Input: expression = "!(f)"
Output: true

Example 2:

Input: expression = "|(f,t)"
Output: true

Example 3:

Input: expression = "&(t,f)"
Output: false

Example 4:

Input: expression = "|(&(t,f,t),!(t))"
Output: false



来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/parsing-a-boolean-expression
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

并且题目描述里很清晰地给出了语法描述，照着写就行：

```
exprList -> "(" expr ("," expr)* ")"
notExpr -> "!" exprList
andExpr -> "&" exprList
orExpr -> "|" exprList
expr -> "t" | "f" | notExpr | andExpr | orExpr
```

### [[中等] 439. 三元表达式解析器](https://leetcode-cn.com/problems/ternary-expression-parser/)

这题的输入是这样的：

```
Example 1:

Input: expression = "T?2:3"
Output: "2"
Explanation: If true, then result is 2; otherwise result is 3.

Example 2:

Input: expression = "F?1:T?4:5"
Output: "4"
Explanation: The conditional expressions group right-to-left. Using parenthesis, it is read/evaluated as:
"(F ? 1 : (T ? 4 : 5))" --> "(F ? 1 : 4)" --> "4"
or "(F ? 1 : (T ? 4 : 5))" --> "(T ? 4 : 5)" --> "4"

Example 3:

Input: expression = "T?T?F:5:3"
Output: "F"
Explanation: The conditional expressions group right-to-left. Using parenthesis, it is read/evaluated as:
"(T ? (T ? F : 5) : 3)" --> "(T ? F : 3)" --> "F"
"(T ? (T ? F : 5) : 3)" --> "(T ? F : 5)" --> "F"

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/ternary-expression-parser
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

语法规则不用多解释了：

```
cond -> "T" | "F"
ternary -> cond "?" expr ":" expr
expr -> cond | Num | ternary
```

不过这样实现是不行的。问题出在`expr`上。直接照着实现的话是这样的：

```typescript
// expr -> cond | Num | ternary
const expr = lazy(() =>
  oneOf(cond, Num, ternary))
```

之前我们说过，目前的parser combinators是没有回溯的，`oneOf`会返回第一个匹配到的结果。在这个语法中，`expr`和`ternary`都可以从`cond`开头，如果在`expr`中`cond`写在`ternary`的前面，那么在原本可以匹配到`ternary`的地方，就会先匹配到`cond`并返回。这可能并不是我们想要的。遇到这种情况，只需要调整相对顺序即可：

```diff
- expr -> cond | Num | ternary
+ expr -> Num | ternary | cond
```

### 基本计算器系列

目前一共四道题，都可以用[上一篇](/2021/08/27/parsing-expressions.html)里解析表达式的方法解决。各题具体用到的token略有区别，总结如下：

| 问题 | 备注 |
| --- | ----------- |
| [[困难] 224. 基本计算器](https://leetcode-cn.com/problems/basic-calculator/) | 空格, +, -, -(前缀), 括号 |
| [[中等] 227. 基本计算器 II](https://leetcode-cn.com/problems/basic-calculator-ii/) | 空格, +, -, *, / |
| [[困难] 772. 基本计算器 III](https://leetcode-cn.com/problems/basic-calculator-iii/) | +, -, *, /, 括号 |
| [[困难] 770. 基本计算器 IV](https://leetcode-cn.com/problems/basic-calculator-iv/) | 空格, +, -, *, 括号 |

只不过在第`IV`题中，解析完表达式才只是第一步而已...

### [[困难] 726. 原子的数量](https://leetcode-cn.com/problems/number-of-atoms/)

这题的输入是这样的：

```
Example 1:

Input: formula = "H2O"
Output: "H2O"
Explanation: The count of elements are {'H': 2, 'O': 1}.

Example 2:

Input: formula = "Mg(OH)2"
Output: "H2MgO2"
Explanation: The count of elements are {'H': 2, 'Mg': 1, 'O': 2}.

Example 3:

Input: formula = "K4(ON(SO3)2)2"
Output: "K4N2O14S4"
Explanation: The count of elements are {'K': 4, 'N': 2, 'O': 14, 'S': 4}.

Example 4:

Input: formula = "Be32"
Output: "Be32"



来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/number-of-atoms
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

直接按照题目描述翻译的话，很容易写出这样的语法：

```
formula -> Atom Count? | formula+ | '(' formula ')' Count?
```

然而这是不行的。原因是`formula -> formula+`这样的产生式是左递归的。直接实现并运行的话会递归调用到栈溢出。这跟手写递归下降算法遇到的左递归问题是一样的(由于我们的parser combinator用到了promise，所以不会栈溢出，而是会卡死直至OOM。这也是少数几个让人意识到JavaScript里的async/await只是个假象的地方)。解决的方法还是改写语法：

```
atomGroup -> Atom Count?
formulaGroup -> "(" formula ")" Count?
formula -> (atomGroup | formulaGroup)*
```

### [[困难] 1096. 花括号展开 II](https://leetcode-cn.com/problems/brace-expansion-ii/)

这题的输入是这样的：

{% raw %}
```
Example 1:

Input: expression = "{a,b}{c,{d,e}}"
Output: ["ac","ad","ae","bc","bd","be"]

Example 2:

Input: expression = "{{a,z},a{b,c},{ab,z}}"
Output: ["a","ab","ac","z"]
Explanation: Each distinct word is written only once in the final answer.

 

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/brace-expansion-ii
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```
{% endraw %}

对于这样的语法我们应该已经比较有经验了：

```
union -> "{" expr ("," expr)+ "}"
expr -> (Letter | union)+
```

不过这题最后一个测试用例是`"{a,b},x{c,{d,e}y}"`，真是恶意满满...

### [[困难] 736. Lisp语法解析](https://leetcode-cn.com/problems/parse-lisp-expression/)

这题的输入是这样的：

```
Example 1:

Input: expression = "(let x 2 (mult x (let x 3 y 4 (add x y))))"
Output: 14
Explanation: In the expression (add x y), when checking for the value of the variable x,
we check from the innermost scope to the outermost in the context of the variable we are trying to evaluate.
Since x = 3 is found first, the value of x is 3.

Example 2:

Input: expression = "(let x 3 x 2 x)"
Output: 2
Explanation: Assignment in let statements is processed sequentially.

Example 3:

Input: expression = "(let x 1 y 2 x (add x y) (add x y))"
Output: 5
Explanation: The first (add x y) evaluates as 3, and is assigned to x.
The second (add x y) evaluates as 3+2 = 5.

Example 4:

Input: expression = "(let x 2 (add (let x 3 (let x 4 x)) x))"
Output: 6
Explanation: Even though (let x 4 x) has a deeper scope, it is outside the context
of the final x in the add-expression.  That final x will equal 2.

Example 5:

Input: expression = "(let a1 3 b2 (add a1 1) b2)"
Output: 4
Explanation: Variable names can contain digits after the first character.



来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/parse-lisp-expression
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
```

按照题目描述也可以很容易写出语法：

```
expr -> Int | letExpr | opExpr | Var
letExpr -> "(" "let" (Var expr)+ expr ")"
opExpr -> "(" Op expr expr ")"
```

不过要完成这题还得实现变量的作用域，这已经开始步入实现一门语言的领域了。

### 小结

这波题目刷下来让我们对parser combinators的能力又有了进一步的认识。我们熟悉了`.map()`要怎么写，如何把现有的parser构造函数组合成更复杂的函数，了解了为什么有时候需要调整语法规则中各部分的顺序，以及左递归的语法虽然写起来很自然但无法被parser combinators所用。关于处理左递归语法有一套规范的完整理论，不过处理完的结果不一定容易理解。要想把左递归语法改写成清晰的消除了左递归的语法，需要一定的练习和经验。

此外我们可以看到，语法分析的问题只要能写出语法，剩下的工作就是机械地把语法翻译成代码而已。听到“机械”一词有没有觉得DNA动了？没错，我们可以从语法规则中自动生成解析代码。可以参考我写的一个parser生成器[1]，其甚至可以从描述语法规则的语法规则中生成自身，在编译器的各个组件中率先完成了自举。

### 参考资料

文中的`between`和`sepBy`参考了`arcsecond`[2]的API。

* [1] [A Bootstrapping Parser Generator](https://github.com/qszhu/parserGen)
* [2] [GitHub - francisrstokes/arcsecond: ✨Zero Dependency Parser Combinator Library for JS Based on Haskell’s Parsec](https://github.com/francisrstokes/arcsecond)