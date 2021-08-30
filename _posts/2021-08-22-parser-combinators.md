---
layout: post
title: 用TypeScript实现一门语言(1)——语法分析
date: 2021-08-22 22:00:00 +0800
tags: [compiler, parser_combinators, functional_programming]
---

### 目录
* 用TypeScript实现一门语言(1)——语法分析 <-- 你在这里
* [用TypeScript实现一门语言(2)——表达式解析](/2021/08/27/parsing-expressions.html)

### 背景

极客时间最近上了一门编译原理实战课《手把手带你写一门编程语言》[1]。课程目录让人看了相当兴奋，比Bob写了多年的新书[10]还多了很多高级主题。尤其是看到课程使用的语言是TypeScript时，我心想这次可能会有点不一样的东西。要知道Bob虽然把编译器相关的教程写得很浅显易懂，但总是喜欢用Java来实现，号称“如果都能用Java实现了，用其他任何语言也就都能实现了”[11]。但Java的语法并不是很简洁，平添了许多噪音，还缺少了很多语言特性。所以在看到前几课那些写得像Java一样的TS代码之后，我有些失望。

此外，课程涉及到的编译原理知识还是我上大学时的那些东西，虽然经典，但着实枯燥。更有甚者，每次扩展一下语法都几乎要从零开始手写词法和语法分析器，连代码生成器都不用，工作量很大。

在这几年的工作中，我逐渐了解到一种方法，在语言诞生初期不断迭代语法规则的时候，不用借助代码生成器就能快速手写语法分析器(`Parser`)，且基本不需要了解编译原理相关的知识。于是我准备来尝试下，用这种方法来实现一门编程语言。

### 定义

首先我们简要回顾下什么是编译器。抽象来看，编译器接收源代码作为输入，再输出编译目标。可以把编译器想象成一个函数，接受一个参数，返回一个值。而编译的各个阶段都可以看成一个个函数，最终的编译器就是这些函数的**组合**。而在编译器的前端中，语法分析器是这样一个函数：接受源代码，返回一棵语法树。

我们先来尝试定义一个分析器函数`ParserFn`：

```typescript
type ParserFn = (state: ParserState) => Promise<ParserState>
```

其会对当前的解析状态`ParserState`做一些操作，然后返回一个新的状态（这里用到了`Promise`只是为了方便处理解析过程中出错的情况）。

解析状态是这样定义的：

```typescript
type ParserState = {
  target: string
  index: number
  result?: unknown
}
```

其中包含了原始的输入`target`，当前的处理位置`index`，以及当前的处理结果`result`。

为了方便使用，我们再定义一个分析器类`Parser`：

```typescript
class Parser {
  constructor(readonly parserFn: ParserFn) {}

  parse(target: string) {
    const initialState = {
      target,
      index: 0,
    }
    return this.parserFn(initialState)
  }
}
```

这样的话，假如我们已经有了一个`Parser`的实例`parser`，就可以直接传入字符串来解析了：

```typescript
const res = await parser.parse('hello')
```

### 第一个实现：`str`

现在假设我们要匹配上面的字符串`'hello'`，或是更通用一点，匹配任意指定的字符串，需要怎么写`Parser`呢？我们可以写这么个构造函数：

```typescript
export const str = (s: string) =>
  new Parser(async (state: ParserState) => {
    const { target, index } = state
    const slicedTarget = target.slice(index)

    if (slicedTarget.length === 0) {
      throw `str: Tried to match "${s}", but got unexpected EOF.`
    }

    // check match
    if (slicedTarget.startsWith(s)) {
      return { ...state, index: index + s.length, result: s }
    }

    throw `str: Tried to match "${s}", but got "${peek(state)}" at index ${index}`
  })

// helper function
function peek(state: ParserState) {
  const { target, index } = state
  return target.slice(index, index + 10)
}
```

其接受一个待匹配的字符串，创建并返回一个`Parser`实例。parser的核心逻辑就是检查一下当前状态中，`target`中从`index`开始的字符串是否与传入的字符串匹配。如果匹配的话，就返回一个更新后的状态（设置了`result`和新的`index`）。剩下的都是对未匹配到的情况的处理，应该挺好理解的吧。

让我们来实际用一下：

```typescript
const parser = str('hello')
{
  const res = await parser.parse('hello')
  // { target: 'hello', index: 5, result: 'hello' }
}
{
  const res = await parser.parse('world')
  // str: Tried to match "hello", but got "world" at index 0
}
```

看到这里，你可能会想，这玩意儿看上去平平无奇。相同的功能我写一行代码就搞定了，为什么要整那么多有的没的出来。不要着急，看下去就知道了。

### 第一个语法

第一课[2]里有个超简单的语言的语法[3]：

```
1. prog = (functionDecl | functionCall)* ;

2. functionDecl: "function" Identifier "(" ")"  functionBody;

3. functionBody : '{' functionCall* '}' ;

4. functionCall : Identifier '(' parameterList? ')' ;

5. parameterList : StringLiteral (',' StringLiteral)* ;
```

从上往下看，第一条规则讲了，一个`prog`由零到多个`functionDecl`或`functionCall`组成。这条规则需要两个parser，一个可以匹配`零到多个`的情况，另一个可以匹配`或`的情况。写成代码的话：

```typescript
const prog = zeroOrMore(oneOf(functionDecl, functionCall))
```

我们将构造这两个parser的函数分别命名为`zeroOrMore`和`oneOf`。

### 实现`zeroOrMore`

我们先来试试`zeroOrMore`：

```typescript
export const zeroOrMore = (parser: Parser) =>
  new Parser(async (state: ParserState) => {
    const results = []
    let nextState = state

    try {
      while (true) {
        nextState = await parser.parserFn(nextState)
        results.push(nextState.result)
      }
    } catch (e) {}

    return { ...nextState, result: results }
  })
```

可以看到其尝试不停地用传入的`parser`去匹配解析状态，直到不能匹配为止，并把期间的匹配结果都收集起来，作为新状态的结果返回。让我们来试一下：

```typescript
const parser = zeroOrMore(str('hello'))
{
  const res = await parser.parse('hellohello')
  // { target: 'hellohello', index: 10, result: [ 'hello', 'hello' ] }
}
{
  const res = await parser.parse('helloworld')
  // { target: 'helloworld', index: 5, result: [ 'hello' ] }
}
{
  const res = await parser.parse('world')
  // { target: 'world', index: 0, result: [] }
}
```

另外注意到`zeroOrMore`是不会抛出错误的，不能匹配到的话只会返回空的结果。

### 实现`oneOf`

再来试试`oneOf`：

```typescript
export const oneOf = (...parsers: Parser[]) =>
  new Parser(async (state: ParserState) => {
    for (const parser of parsers) {
      try {
        return await parser.parserFn(state)
      } catch (e) {}
    }
    throw `oneOf: Unable to match any parser at index ${state.index}: "${peek(state)}"`
  })
```

这里构造的parser会返回传入的一连串parser中第一个匹配成功的结果：

```typescript
const parser = oneOf(str('hello'), str('world'))
{
  const res = await parser.parse('helloworld')
  // { target: 'helloworld', index: 5, result: 'hello' }
}
{
  const res = await parser.parse('你好世界')
  // oneOf: Unable to match any parser at index 0: "你好世界"
}
```

### 合体！

把这两个parser**组合**起来的话，就可以完成这样的事情：

```typescript
const parser = zeroOrMore(oneOf(str('hello'), str('world')))
{
  const res = await parser.parse('helloworld')
  // { target: 'helloworld', index: 10, result: [ 'hello', 'world' ] }
}
{
  const res = await parser.parse('worldhello')
  // { target: 'worldhello', index: 10, result: [ 'world', 'hello' ] }
}
```

这里你可以停下来想一下，如果不用上面定义的`Parser`的话，你会怎么实现？

### 实现`seqOf`

再来看第二条规则：

```
functionDecl: "function" Identifier "(" ")"  functionBody ;
```

这条是说，一个`functionDecl`是由一个`"function"`关键字，一个标识符，一个左括号，一个右括号和一个`functionBody`组成的。这里我们需要构造能匹配一个`序列`的parser：

```typescript
export const seqOf = (...parsers: Parser[]) =>
  new Parser(async (state: ParserState) => {
    const results = []
    let nextState = state

    for (const parser of parsers) {
      nextState = await parser.parserFn(nextState)
      results.push(nextState.result)
    }

    return { ...nextState, result: results }
  })
```

这个parser的实现跟上面的`zeroOrMore`有些类似，不同的地方在于传入的parser必须依次匹配成功，不然就会抛出错误：

```typescript
const parser = seqOf(str('hello'), str('world'))
{
  const res = await parser.parse('helloworld')
  // { target: 'helloworld', index: 10, result: [ 'hello', 'world' ] }
}
{
  const res = await parser.parse('hello world')
  // str: Tried to match "world", but got " world" at index 5
}
```

这样我们就可以写出：

```typescript
const functionDecl = seqOf(str('function'), Identifier, str('('), str(')'), functionBody)
```

来看第三条规则：

```
functionBody : '{' functionCall* '}' ;
```

可以直接写出来了：

```typescript
const functionBody = seqOf(str('{'), zeroOrMore(functionCall), str('}'))
```

### 实现`zeroOrOne`

再来看第四条规则：

```
functionCall : Identifier '(' parameterList? ')' ;
```

这里需要能构建一个新的parser，来匹配`0到1个`的`parameterList`：

```typescript
export const zeroOrOne = (parser: Parser) =>
  new Parser(async (state: ParserState) => {
    try {
      const nextState = await parser.parserFn(state)
      return { ...nextState, result: nextState.result }
    } catch (e) {}

    return { ...state, result: undefined }
  })
```

和上面的`zeroOrMany`一样，`zeroOrOne`没有匹配的时候也不会抛出错误：

```typescript
const parser = zeroOrOne(str('hello'))
{
  const res = await parser.parse('hello')
  // { target: 'hello', index: 0, result: 'hello' }
}
{
  const res = await parser.parse('world')
  // { target: 'world', index: 0, result: undefined }
}
```

这样第四条规则也可以写出来了：

```typescript
const functionCall = seqOf(Identifier, str('('), zeroOrOne(parameterList), str(')'))
```

最后是第五条规则：

```
parameterList : StringLiteral (',' StringLiteral)* ;
```

也可以直接写出来了：

```typescript
const parameterList = seqOf(StringLiteral, zeroOrMore(seqOf(str(','), StringLiteral)))
```

这样所有的语法规则都变成代码实现了：

```typescript
const parameterList = seqOf(StringLiteral, zeroOrMore(seqOf(str(','), StringLiteral)))
const functionCall = seqOf(Identifier, str('('), zeroOrOne(parameterList), str(')'))
const functionBody = seqOf(str('{'), zeroOrMore(functionCall), str('}'))
const functionDecl = seqOf(str('function'), Identifier, str('('), str(')'), functionBody)
const prog = zeroOrMore(oneOf(functionDecl, functionCall))
```

注意到代码的顺序与上面规则的顺序是相反的，是因为在`strict`模式下，`TypeScript`不允许使用未声明的变量，按照规则顺序写的话会编译错误。然而并不是所有的语法规则都只要倒过来实现就行的，后面我们会看到。

### 实现`regExp`

即便如此，现在编译上面的代码的话还是会失败的，因为我们还没有定义规则里的终结符`Identifier`和`StringLiteral`。`Identifier`的词法规则是这样的[4]：

```
Identifier: [a-zA-Z_][a-zA-Z0-9_]* ;
```

虽然这里也可以用上面定义的那些parser构造函数写出来，但因为本身借用了正则表达式的语法，用正则表达式来实现会更方便。于是我们定义一个支持正则匹配的parser：

```typescript
export const regExp = (pattern: RegExp) =>
  new Parser(async (state: ParserState) => {
    console.assert(pattern.source.startsWith('^'), 'regExp should start with "^"')

    const { target, index } = state

    const slicedTarget = target.slice(index)
    if (slicedTarget.length === 0) {
      throw `regExp: Tried to match "${pattern}", but got unexpected EOF.`
    }

    const match = slicedTarget.match(pattern)
    if (match) {
      const [result] = match
      return { ...state, index: index + result.length, result }
    }

    throw `regExp: Tried to match "${pattern}", but got "${peek(state)}" at index ${index}`
  })
```

这里的实现跟`str`差不多。需要注意的是如果正则表达式不是以`^`开头的话，则可能在某个中间位置匹配到结果，那样就相当于跳过了某些字符做了匹配。这在大部分情况下都是错误的。开头的断言就是用来检查这个情况的：

```typescript
console.assert(pattern.source.startsWith('^'), 'regExp should start with "^"')
```

让我们来测试一下：

```typescript
const Identifier = regExp(/^[a-zA-Z_][a-zA-Z0-9_]*/)
{
  const res = await Identifier.parse('hello_123')
  // { target: 'hello_123', index: 9, result: 'hello_123' }
}
{
  const res = await Identifier.parse('01world')
  // regExp: Tried to match "/^[a-zA-Z_][a-zA-Z0-9_]*/", but got "01world" at index 0
}
```

`StringLiteral`的规则课程里没有给出来。我是这样写的：

```typescript
const StringLiteral = regExp(/^"[^"]*"/)
```

当然这样就只支持双引号(")，单引号(')和反引号(`)就不支持了。你也可以尝试自己加一下。

### `Parser Combinators`

这样我们就把语法规则和词法规则翻译成了一系列parser的 **“组合”**。这种实现模式有个名字，叫`combinator`模式[5]。而parser的组合就称为`parser combinators`（有译作分析器组合子）。这样实现的话，规则和代码有着一目了然的对应关系：

```typescript
const StringLiteral = regExp(/^"[^"]*"/)

// Identifier: [a-zA-Z_][a-zA-Z0-9_]* ;
const Identifier = regExp(/^[a-zA-Z_][a-zA-Z0-9_]*/)

// parameterList : StringLiteral (',' StringLiteral)* ;
const parameterList = seqOf(StringLiteral, zeroOrMore(seqOf(str(','), StringLiteral)))

// functionCall : Identifier '(' parameterList? ')' ;
const functionCall = seqOf(Identifier, str('('), zeroOrOne(parameterList), str(')'))

// functionBody : '{' functionCall* '}' ;
const functionBody = seqOf(str('{'), zeroOrMore(functionCall), str('}'))

// functionDecl: "function" Identifier "(" ")"  functionBody;
const functionDecl = seqOf(str('function'), Identifier, str('('), str(')'), functionBody)

// prog = (functionDecl | functionCall)* ;
const prog = zeroOrMore(oneOf(functionDecl, functionCall))
```

而且我们好像连词法分析器都没写。如果把上面那些parser构造函数放到一个库文件里，那么每次要实现一个新语言的词法规则和语法规则时，只需要写跟这些规则的数量相当的代码就可以完成语法分析了。要知道代码越少越简单意味着维护成本越低，这些每一个都看上去平平无奇的代码真有那么大的威力吗？让我们立马测试下。

先试试一个最简单的函数调用：

```typescript
const res = await prog.parse('foo()')
/*
{
  target: 'foo()',
  index: 5,
  result: [ [ 'foo', '(', undefined, ')' ] ]
}
*/
```

可以看到匹配结果，不过缺少了一些结构。让我们对结果做些变换令其成为语法树。

### 实现`map`

在`Parser`类中添加如下方法：

```typescript
class Parser {
  // ...

  map(fn: (arg: any) => unknown) {
    return new Parser(async (state) => {
      const nextState = await this.parserFn(state)
      return { ...nextState, result: fn(nextState.result) }
    })
  }}
}
```

注意到其用传入的函数构造了一个能对当前匹配结果做变换的`Parser`，因此其返回值仍能被用于之前的parser组合中。

然后为上面匹配到的规则添加变换：

```typescript
// ...

// functionCall : Identifier '(' parameterList? ')' ;
const functionCall =
  seqOf(Identifier, str('('), zeroOrOne(parameterList), str(')'))
    .map(([name, _lparen, params, _rparen]) => ({ type: 'functionCall', name, params }))

// ...

// prog = (functionDecl | functionCall)* ;
const prog =
  zeroOrMore(oneOf(functionDecl, functionCall))
    .map((stmts) => ({ type: 'prog', stmts }))
```

再来试一下：

```typescript
const { result } = await prog.parse('foo()')
/*
{
  "type": "prog",
  "stmts": [
    {
      "type": "functionCall",
      "name": "foo"
    }
  ]
}
*/
```

是不是开始有点意思了？你可以试着把剩下的规则匹配到的结果也变换下，让最终结果成为一颗语法树。

### 实现`parseToEnd`

让我们来试试带参数的函数调用：

```typescript
const res = await prog.parse('println("hello", "world")')
/*
{
  "target": "println(\"hello\", \"world\")",
  "index": 0,
  "result": {
    "type": "prog",
    "stmts": []
  }
}
*/
```

Emmm...虽然没有抛出错误，但我们并没有得到想要的结果。注意到解析状态中`index`为0，说明解析器没有匹配到任何结果。我们把在解析过程结束后仍有未匹配到输入的情况定义为出错的情况。为了处理这种情况，在`Parser`类中添加如下方法：

```typescript
class Parser {
  // ...

  async parseToEnd(target: string) {
    const res = await this.parse(target)
    if (res.index !== target.length) {
      throw `Parsing end at ${res.index}: "${peek(res)}"`
    }
    return res
  }
}
```

这样遇到上面的情况就会报错：

```typescript
const res = await prog.parseToEnd('println("hello", "world")')
// Parsing end at 0: "println("h"
```

### 错误调试

好了，接下来我们就要分析为什么会出错了。`prog` parser是指望不上了。按照规则，前一个匹配上的应该是`functionCall`，让我们来试一下：

```typescript
const res = await functionCall.parseToEnd('println("hello", "world")')
// str: Tried to match ")", but got ", "world")" at index 15
```

看起来我们的parser在遇到“`,`”之前就以为参数列表已经结束了，想要匹配“`)`”，而下一个字符却是“`,`”。说明在用`parameterList`匹配`"hello", "world"`时出了问题：

```typescript
const res = await parameterList.parseToEnd('"hello", "world"')
// Parsing end at 7: ", "world""
```

还是只匹配到了第一个参数，说明`(',' StringLiteral)*`这部分规则没能匹配到`, "world"`：

```typescript
const parser = zeroOrMore(seqOf(str(','), StringLiteral)) 
const res = await parser.parseToEnd(', "world"')
// Parsing end at 0: ", "world""
```

还是不行。再剥一层：

```typescript
const parser = seqOf(str(','), StringLiteral) 
const res = await parser.parseToEnd(', "world"')
// regExp: Tried to match "/^"[^"]*"/", but got " "world"" at index 1
```

这下明白了，“`,`”和字符串之间有个空格没办法匹配。因为通常编程语言会允许在代码中加入任意的空白符来增加（或是破坏[6]）可读性，所以这些空白符也是要能被匹配出来的。先定义一个能匹配空白符的parser构造函数：

```typescript
const whitespace = regExp(/^\s*/)
```

再把`str`和`regExp`包装一下。如果遇到前面有空白符号，则在匹配之后丢弃：

```typescript
const token = (s: string) =>
  seqOf(whitespace, str(s))
    .map(([_, res]) => res)

const regexToken = (pattern: RegExp) =>
  seqOf(whitespace, regExp(pattern))
    .map(([_, res]) => res)
```

最后把原来规则对应的代码里所有的`str`替换成`token`，`regExp`替换成`regexToken`。再试一下：

```typescript
const { result } = await prog.parseToEnd('println("hello", "world")')
/*
{
  "type": "prog",
  "stmts": [
    {
      "type": "functionCall",
      "name": "println",
      "params": [
        "\"hello\"",
        "\"world\""
      ]
    }
  ]
}
*/
```

终于成功了。源代码中还需要被忽略的是各种注释，你可以试着实现下。

### 更复杂的例子

来试下第一课最后的例子：

```typescript
const source = `
function foo() {
  println("in foo...")
}

function bar() {
  println("in bar...")
  foo()
}

bar()`
const { result } = await prog.parseToEnd(source)
/*
{
  "type": "prog",
  "stmts": [
    {
      "type": "functionDecl",
      "name": "foo",
      "body": [
        {
          "type": "functionCall",
          "name": "println",
          "params": [
            "\"in foo...\""
          ]
        }
      ]
    },
    {
      "type": "functionDecl",
      "name": "bar",
      "body": [
        {
          "type": "functionCall",
          "name": "println",
          "params": [
            "\"in bar...\""
          ]
        },
        {
          "type": "functionCall",
          "name": "foo"
        }
      ]
    },
    {
      "type": "functionCall",
      "name": "bar"
    }
  ]
}
*/
```

是不是已经可以直接拿来做语义分析了？

### 实现`lazy`

让我们来试试更复杂的语法[7]，注意到其中包含了递归的规则：

```
...
expression: primary (binOP primary)* ;
primary: StringLiteral | DecimalLiteral | IntegerLiteral | functionCall | '(' expression ')' ;
...
```

按照之前的实现方式，如果先实现`expression`再实现`primary`，编译器就会报错说在`primary`在定义前就被`expression`使用了；如果先实现`primary`再实现`expression`，则是`expression`在定义前就被`primary`使用了。无论我们如何调整这些规则实现的先后顺序，都会遇到“在定义前使用”的编译错误。这可以通过惰性求值来避免：

```typescript
export const lazy = (thunk: () => Parser) =>
  new Parser((state: ParserState) => {
    const parser = thunk()
    return parser.parserFn(state)
  })
```

给之前的语法规则实现都套上`lazy`，就可以按顺序实现了：

```typescript
// 1. prog = (functionDecl | functionCall)* ;
const prog = lazy(() =>
  zeroOrMore(oneOf(functionDecl, functionCall)))
    .map((stmts) => ({ type: 'prog', stmts }))

// 2. functionDecl: "function" Identifier "(" ")"  functionBody;
const functionDecl = lazy(() =>
  seqOf(token('function'), Identifier, token('('), token(')'), functionBody))
    .map(([_, name, _lp, _rp, body]) => ({ type: 'functionDecl', name, body }))

// 3. functionBody : '{' functionCall* '}' ;
const functionBody = lazy(() =>
  seqOf(token('{'), zeroOrMore(functionCall), token('}')))
    .map(([_lb, calls, _rb]) => calls)

// 4. functionCall : Identifier '(' parameterList? ')' ;
const functionCall = lazy(() =>
  seqOf(Identifier, token('('), zeroOrOne(parameterList), token(')')))
    .map(([name, _lp, params, _rp]) => ({ type: 'functionCall', name, params }))

// 5. parameterList : StringLiteral (',' StringLiteral)* ;
const parameterList = lazy(() =>
  seqOf(StringLiteral, zeroOrMore(seqOf(token(','), StringLiteral))))
    .map(([param, params]) => [param, ...params.map(([_comma, param]: unknown[]) => param)])
```

你可以自行验证下这跟之前的实现是等价的。

接下来你就可以尝试实现更复杂的语法[8]了。实现过程中你可能会发现很多小问题，有机会的话下次我们就来讲讲这些问题。

### 参考资料

文中parser combinators的实现参考了`arcsecond`[9]的部分API和实现。

* [1] [手把手带你写一门编程语言_编程语言_手把手_编译原理_编译技术_运行时_汇编语言_宫文学_后端_虚拟机_字节码_TS_TypeScript_鸿蒙-极客时间](https://time.geekbang.org/column/intro/436)
* [2] [01｜实现一门超简单的语言最快需要多久？-极客时间](https://time.geekbang.org/column/article/406179)
* [3] [01/play.ts · RichardGong/Craft A Language - Gitee.com](https://gitee.com/richard-gong/craft-a-language/blob/master/01/play.ts)
* [4] [02/play.ts · RichardGong/Craft A Language - Gitee.com](https://gitee.com/richard-gong/craft-a-language/blob/master/02/play.ts)
* [5] [Combinator pattern - HaskellWiki](https://wiki.haskell.org/Combinator_pattern)
* [6] [winner/prog.c at main · ioccc-src/winner](https://github.com/ioccc-src/winner/blob/main/2020/tsoj/prog.c)
* [7] [04/parser.ts · RichardGong/Craft A Language - Gitee.com](https://gitee.com/richard-gong/craft-a-language/blob/master/04/parser.ts)
* [8] [06/parser.ts · RichardGong/Craft A Language - Gitee.com](https://gitee.com/richard-gong/craft-a-language/blob/master/06/parser.ts)
* [9] [GitHub - francisrstokes/arcsecond: ✨Zero Dependency Parser Combinator Library for JS Based on Haskell’s Parsec](https://github.com/francisrstokes/arcsecond)
* [10] [Table of Contents · Crafting Interpreters](https://craftinginterpreters.com/contents.html)
* [11] [Pratt Parsers: Expression Parsing Made Easy              – journal.stuffwithstuff.com](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)