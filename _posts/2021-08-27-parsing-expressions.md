---
layout: post
title: 用TypeScript实现一门语言(2)——表达式解析 
date: 2021-08-27 22:00:00 +0800
tags: [compiler, parser_combinators, functional_programming, expression]
---

### 目录
* [用TypeScript实现一门语言(1)——语法分析](/2021/08/22/parser-combinators.html)
* 用TypeScript实现一门语言(2)——表达式解析 <-- 你在这里

### 背景

在[上一篇](/2021/08/22/parser-combinators.html)中，我们实现了一个简单的parser combinators库，并用其解析了一个简单的语法。最后我们说要挑战一下更复杂的语法[1]。我不知道你怎么样，反正我啪的一下就写好了，很快啊（才怪）：

```typescript
// prog = statementList* EOF;
const prog = lazy(() =>
  zeroOrMore(statementList))

// statementList = (variableDecl | functionDecl | expressionStatement)+ ;
const statementList = lazy(() =>
  oneOrMore(oneOf(variableDecl, functionDecl, expressionStatement)))

// statement: block | expressionStatement | returnStatement | ifStatement | forStatement 
//         | emptyStatement | functionDecl | variableDecl ;
const statement = lazy(() =>
  oneOf(block, expressionStatement, returnStatement, ifStatement, forStatement,
    emptyStatement, functionDecl, variableDecl))

// block : '{' statementList? '}' ;
const block = lazy(() =>
  seqOf(token('{'), zeroOrOne(statementList), token('}')))

// ifStatement : 'if' '(' expression ')' statement ('else' statement)? ;
const ifStatement = lazy(() => 
  seqOf(token('if'), token('('), expression, token(')'), statement,
    zeroOrOne(seqOf(token('else'), statement))))

// forStatement : 'for' '(' (expression | 'let' variableDecl)? ';' expression? ';' expression? ')' statement ;
const forStatement = lazy(() =>
  seqOf(token('for'), token('('),
    zeroOrOne(oneOf(expression, seqOf(token('let'), variableDecl))), token(';'),
    zeroOrOne(expression), token(';'),
    zeroOrOne(expression), token(')'), statement))

// variableStatement : 'let' variableDecl ';';
const variableStatement = lazy(() =>
  seqOf(token('let'), variableDecl, token(';')))

// variableDecl : Identifier typeAnnotation？ ('=' expression)? ;
const variableDecl = lazy(() =>
  seqOf(Identifier, zeroOrOne(typeAnnotation), zeroOrOne(seqOf(token('='), expression))))

// typeAnnotation : ':' Identifier;
const typeAnnotation = lazy(() =>
  seqOf(token(':'), Identifier))

// functionDecl: "function" Identifier callSignature  block ;
const functionDecl = lazy(() =>
  seqOf(token('function'), Identifier, callSignature, block))

// callSignature: '(' parameterList? ')' typeAnnotation? ;
const callSignature = lazy(() =>
  seqOf(token('('), zeroOrOne(parameterList), token(')'), zeroOrOne(typeAnnotation)))

// returnStatement: 'return' expression? ';' ;
const returnStatement = lazy(() =>
  seqOf(token('return'), zeroOrOne(expression), token(';')))

// emptyStatement: ';' ;
const emptyStatement = lazy(() =>
  token(';'))

// expressionStatement: expression ';' ;
const expressionStatement = lazy(() =>
  seqOf(expression, token(';')))

// expression: assignment;
const expression = lazy(() =>
  assignment)

// assignment: binary (assignmentOp binary)* ;
const assignment = lazy(() =>
  seqOf(binary, zeroOrMore(seqOf(assignmentOp, binary))))

// binary: unary (binOp unary)* ;
const binary = lazy(() =>
  seqOf(unary, zeroOrMore(seqOf(binOp, unary))))

// unary: primary | prefixOp unary | primary postfixOp ;
const unary = lazy(() =>
  oneOf(primary, seqOf(prefixOp, unary), seqOf(primary, postfixOp)))

// primary: StringLiteral | DecimalLiteral | IntegerLiteral | functionCall | '(' expression ')' ;
const primary = lazy(() =>
  oneOf(StringLiteral, DecimalLiteral, IntegerLiteral, functionCall, seqOf(token('('), expression, token(')'))))

// assignmentOp = '=' | '+=' | '-=' | '*=' | '/=' | '>>=' | '<<=' | '>>>=' | '^=' | '|=' ;
const assignmentOp = oneOf(...['=', '+=', '-=', '*=', '/=', '>>=', '<<=', '>>>=', '^=', '|='].map(token))
  
// binOp: '+' | '-' | '*' | '/' | '==' | '!=' | '<=' | '>=' | '<'
//      | '>' | '&&'| '||'|...;
const binOp = oneOf(...['+', '-', '*', '/', '==', '!=', '<=', '>=', '<', '>', '&&', '||'].map(token))

// prefixOp = '+' | '-' | '++' | '--' | '!' | '~';
const prefixOp = oneOf(...['+', '-', '++', '--', '!', '~'].map(token))

// postfixOp = '++' | '--'; 
const postfixOp = oneOf(...['++', '--'].map(token))

// functionCall : Identifier '(' argumentList? ')' ;
const functionCall = lazy(() =>
  seqOf(Identifier, token('('), zeroOrOne(argumentList), token(')')))

// argumentList : expression (',' expression)* ;
const argumentList = lazy(() =>
  seqOf(expression, zeroOrMore(seqOf(token(','), expression))))
```

注意上面的代码中省略了终结符和对匹配结果的`map`。不过这时编译器还有报错，说`callSignature`中的`parameterList`未定义。看起来是漏掉了，我们拿前几课里的补上：

```typescript
// parameterList : parameter (',' parameter)* ;
const parameterList = lazy(() =>
  seqOf(parameter, zeroOrMore(seqOf(token(','), parameter))))

// parameter : Identifier typeAnnotation? ; 
const parameter = lazy(() =>
  seqOf(Identifier, zeroOrOne(typeAnnotation)))
```

让我们来试试简单的变量声明语句：

```typescript
const source = `let a: number = 1 + 2 + 3;`
const res = await prog.parseToEnd(source)
// Parsing end at 0: "let a: number = 1 + 2 + 3;"
```

结果出错了。用我们上次讲过的调试方法，发现在应该用`variableStatement`匹配的地方用了`variableDecl`去匹配。如果你的`tsconfig.json`设置了`noUnusedLocals: true`，就会发现编译器提示`variableStatement`未使用。我们来修改一下：

```typescript
// statementList = (variableStatement | functionDecl | expressionStatement)+ ;
const statementList = lazy(() =>
  oneOrMore(oneOf(variableStatement, functionDecl, expressionStatement)))
```

再次运行：

```typescript
const source = `let a: number = 1 + 2 + 3;`
const res = await prog.parseToEnd(source)
/*
{ type: 'prog',
  stmts:
   [ { type: 'let',
       decl:
        { name: 'a', type: 'number', val: [ 1, [ '+', 2 ], [ '+', 3 ] ] } } ] }
*/
```

看上去似乎解析出来了。不过再让我们试试修改一下表达式：

```typescript
const source = `let a: number = 1 + 2 * 3;`
const res = await prog.parseToEnd(source)
/*
{ type: 'prog',
  stmts:
   [ { type: 'let',
       decl:
        { name: 'a', type: 'number', val: [ 1, [ '+', 2 ], [ '*', 3 ] ] } } ] }
*/
```

可以看到在解析出来的表达式里，运算符没有优先级的概念。课上[2]这样做没有问题，因为已经先从语法上区分了前缀和后缀表达式：

```
binary: unary (binOp unary)* ;
unary: primary | prefixOp unary | primary postfixOp ;
```

剩下的中缀表达式再用运算符优先级算法构造AST。我们这里也可以这样做，不过那样的话我们的parser就只是完成了词法分析的工作，仅仅获取到了表达式中的token，并且需要在中缀表达式语法对应的`map`里实现运算符优先级算法。有没有更优雅的方法呢？

### 运算符优先级

确保运算符优先级的传统方法之一是修改语法[3]，比如要写出支持`+`和`*`的表达式，语法可以这样写：

```
sum : product ('+' product)*
product : primary ('*' primary)*
primary : Num
```

对应的实现：

```typescript
// sum : product ('+' product)*
const sum = lazy(() =>
  seqOf(product, zeroOrMore(seqOf(token('+'), product))))
    .map(([lhs, rest]) =>
      [lhs, ...rest].reduce((lhs, [op, rhs]) => [lhs, op, rhs]))

// product : primary ('*' primary)*
const product = lazy(() =>
  seqOf(primary, zeroOrMore(seqOf(token('*'), primary))))
    .map(([lhs, rest]) =>
      [lhs, ...rest].reduce((lhs, [op, rhs]) => [lhs, op, rhs]))

// primary : Num
const primary = lazy(() => Num)

// Num: '0' | [1-9] [0-9]* ;
const Num =
  oneOf(token('0'), regexToken(/^[1-9][0-9]*/))
    .map(Number)
```

注意到这里用了`reduce`来将匹配到的结果左结合。比如匹配到的结果是`[1, ['+', 2], ['+', 3]]`，`reduce`之后：

```typescript
> [1, ['+', 2], ['+', 3]].reduce((lhs, [op, rhs]) => [lhs, op, rhs])
[ [ 1, '+', 2 ], '+', 3 ]
```

再来试一下上面的例子：

```typescript
{
  const { result } = await sum.parseToEnd('1 + 2 + 3')
  // [ [ 1, '+', 2 ], '+', 3 ]
}
{
  const { result } = await sum.parseToEnd('1 + 2 * 3')
  // [ 1, '+', [ 2, '*', 3 ] ]
}
```

可以看到能够按照运算符优先级解析出来了。

### 右结合

注意到上面例子里的运算符都是左结合的，如果是右结合的运算符要怎么写呢？比如求幂运算`**`。因为其优先级比`*`要高，所以要插在`*`下面：

```
sum : ...
product : power ('*' power)*
power : primary '**' power | primary
primary : ...
```

对应的实现：

```typescript
// product : power ('*' power)*
const product = lazy(() =>
  seqOf(power, zeroOrMore(seqOf(token('*'), power))))

// power : primary '**' power | primary
const power = lazy(() =>
  oneOf(seqOf(primary, token('**'), power), primary))
```

试一下：

```typescript
const { result } = await sum.parseToEnd('1 ** 2 ** 3')
// [ 1, '**', [ 2, '**', 3 ] ]
const { result } = await sum.parseToEnd('1 + 2 * 3 ** 4 + 5')
//[ [ 1, '+', [ 2, '*', [ 3, '**', 4 ] ] ], '+', 5 ]
```

至此我们对左结合和右结合运算符就都可以处理了。

### 实现简化

虽然按照上面的方式已经可以处理中缀表达式了，但缺点是每遇到一个运算符就要写一个语法规则出来。看一下课上的语法中定义的二元运算符，瞬间头大：

```
binOp: '+' | '-' | '*' | '/' | '==' | '!=' | '<=' | '>=' | '<' | '>' | '&&' | '||' | ... ;
```

首先可以简化的地方是，相同优先级的运算符可以放在一个规则里，比如：

```typescript
// product : power (('*' | '/' | '%') power)*
const product = lazy(() =>
  seqOf(power, zeroOrMore(seqOf(oneOf(token('*'), token('/'), token('%')), power))))
```

但即便这样，有多少种不同的优先级就要写多少规则，而我已经想不出来更多语法规则的名字了 :-(

### reduce和curry化

让我们再来看下最初的语法规则：

```
sum : product ('+' product)*
product : primary ('*' primary)*
primary : Num
```

不知道你有没有看出其中蕴含的模式？没有的话也没关系，让我们来改写一下让其变得更明显：

```
1. sum = f(product, '+')
2. product = f(primary, '*')
3. primary = Num
```

`3.`代入`2.`：

```
1. sum = f(product, '+')
4. product = f(Num, '*')
```

`4.`代入`1.`：

```
sum = f(f(Num, '*'), '+')
```

这样是不是就看着有些眼熟？换种写法：

```
sum = ['*', '+'].reduce((term, op) => f(term, op), Num)
```

也就是说我们只要把运算符按照优先级从高到低放在一个数组里，再`reduce`一下就好了。计算机科学中两大难题之一[4]的“起个好名字”，就这样被淹没在编译器为你准备好的临时变量里了！

于是现在问题就变成上面的`f`函数要怎么写。其函数签名需要接受下一层的语法规则和一个运算符。我们来尝试下：

```typescript
const infix = (nextTerm: Parser, operator: Parser) =>
  seqOf(nextTerm, zeroOrMore(seqOf(operator, nextTerm)))
```

跟上面的规则实现对比下：

```typescript
const sum = ...
  seqOf(product, zeroOrMore(seqOf(token('+'), product))))

const infix = (nextTerm: Parser, operator: Parser) =>
  seqOf(nextTerm, zeroOrMore(seqOf(operator, nextTerm)))
```

可以看到我们只是把下一层的规则和运算符作为函数的参数传入，替换掉具体的规则和运算符而已。让我们尝试把上面的`sum`/`product`等几个规则用这个函数改写下：

```typescript
const product = infix(primary, token('*'))
const sum = infix(product, token('+'))
```

你可以自行验证下这跟之前的实现是等价的。让我们接着试试`reduce`：

```typescript
const expr = [infix(?, oneOf(token('*'), ...))].reduce(...)
```

这里我们就遇到了问题。函数的第一个参数需要接受下一层的规则，但下一层的规则是要`reduce`出来的。怎么办呢？了解一点函数式编程的话这都不是个事儿，curry化一下就好了：

```typescript
const infix = (operator: Parser) => (nextTerm: Parser) =>
  seqOf(nextTerm, zeroOrMore(seqOf(operator, nextTerm)))
```

这样就可以实现了：

```typescript
const expr = [
  infix(oneOf(token('*'), token('/'), token('%'))),
  infix(oneOf(token('+'), token('-')))
].reduce((term, op) => op(term), primary)

{
  const { result } = await expr.parseToEnd('1 + 2 + 3')
  // [ [ 1, '+', 2 ], '+', 3 ]
}
{
  const { result } = await expr.parseToEnd('1 + 2 * 3')
  // [ 1, '+', [ 2, '*', 3 ] ]
}
```

你也可以自己试下，看看是不是跟之前的实现等价。

### 简化右结合运算符

同样，右结合的运算符也可以这样简化：

```typescript
const infixRight = (operator: Parser) => (nextTerm: Parser) => {
  const parser = lazy(() =>
    oneOf(seqOf(nextTerm, operator, parser), nextTerm))
  return parser
}
```

对比一下前面的实现：

```typescript
const power = ...
  oneOf(seqOf(primary, token('**'), power), primary))
```

可以看到也只是抽出了两个参数而已。让我们一起测试下：

```typescript
const exprOps = [
  infixRight(oneOf(token('**'))),
  infix(oneOf(token('*'), token('/'), token('%'))),
  infix(oneOf(token('+'), token('-')))
]

const expr = exprOps.reduce((term, op) => op(term), primary)

{
  const { result } = await expr.parseToEnd('1 ** 2 ** 3')
  // [ 1, '**', [ 2, '**', 3 ] ]
}
{
  const { result } = await expr.parseToEnd('1 + 2 * 3 ** 4 + 5')
  // [ [ 1, '+', [ 2, '*', [ 3, '**', 4 ] ] ], '+', 5 ]
}
```

这样，增加新的运算符和调整运算符优先级，都只要修改其中的`exprOps`数组就可以了。

### 前缀和后缀表达式

上面我们都在处理中缀表达式。现在我们有了合适的轮子，能不能连前缀和后缀表达式一起处理了呢？当然也是可以的。前缀运算符需要向右结合，后缀运算符需要向左结合，所以分别参照中缀表达式中左结合和右结合运算符的实现修改即可。你可以对比一下它们的异同。

后缀和左结合：

```typescript
const postfix = (operator: Parser) => (nextTerm: Parser) =>
  seqOf(nextTerm, zeroOrMore(operator))

const infix = (operator: Parser) => (nextTerm: Parser) =>
  seqOf(nextTerm, zeroOrMore(seqOf(operator, nextTerm)))
```

前缀和右结合：

```typescript
const prefix = (operator: Parser) => (nextTerm: Parser) => {
  const parser = lazy(() =>
    oneOf(seqOf(operator, parser), nextTerm))
  return parser
}

const infixRight = (operator: Parser) => (nextTerm: Parser) => {
  const parser = lazy(() =>
    oneOf(seqOf(nextTerm, operator, parser), nextTerm))
  return parser
}
```

让我们测试一下：

```typescript
const exprOps = [
  postfix(oneOf(token('++'), token('--'))),
  prefix(oneOf(token('++'), token('--'), token('+'), token('-'))),
  infixRight(token('**')),
  infix(oneOf(token('*'), token('/'), token('%'))),
  infix(oneOf(token('+'), token('-'))),
]

{
  const { result } = await expr.parseToEnd('1++')
  // [ 1, '++' ]
}
{
  const { result } = await expr.parseToEnd('++1')
  // [ '++', 1 ]
}
{
  const { result } = await expr.parseToEnd('++1++')
  // [ '++', [ 1, '++' ] ]
}
{
  const { result } = await expr.parseToEnd('1++++')
  // [ [ 1, '++' ], '++' ]
}
{
  const { result } = await expr.parseToEnd('1+++2*3++**++4++')
  // [ [ 1, '++' ], '+', [ 2, '*', [ [ 3, '++' ], '**', [ '++', [ 4, '++' ] ] ] ] ]
}
```

### 括号

让我们一鼓作气，把带括号的表达式也处理了。括号在表达式里有最高优先级，不用按运算符去处理，直接跟终结符放在一起就好：

```typescript
// primary : Num | '(' expr ')'
const primary: Parser = lazy(() =>
  oneOf(Num, seqOf(token('('), expr, token(')'))))
    .map(res => {
      if (res.length === 3) return res[1]
      return res
    })

{
  const { result } = await expr.parseToEnd('((1 + 2) * 3) ** 4')
  // [ [ [ 1, '+', 2 ], '*', 3 ], '**', 4 ]
}
{
  const { result } = await expr.parseToEnd('-1**2')
  // [ [ '-', 1 ], '**', 2 ]
}
{
  const { result } = await expr.parseToEnd('-(1**2)')
  // [ '-', [ 1, '**', 2 ] ]
}
```

`JavaScript`看了直摇头[5]。

### 确定性

到目前为止我们写的parser能支持的语法都是无二义性的。什么意思呢？让我们看个例子，解析表达式`1++++2`会是什么结果：

```typescript
{
  const { result } = await expr.parseToEnd('1++++2')
  // Parsing end at 5: "2"
}
{
  const { result } = await expr.parseToEnd('1+ +++2')
  // [ 1, '+', [ '++', [ '+', 2 ] ] ]
}
{
  const { result } = await expr.parseToEnd('1++ ++2')
  // Parsing end at 6: "2"
}
{
  const { result } = await expr.parseToEnd('1+++ +2')
  // [ [ 1, '++' ], '+', [ '+', 2 ] ]
}
```

可以看到直接解析的话是会报错的。难道就不能解析成`1+ +++2`或`1+++ +2`的结果吗？如果相同的表达式能解析出不同的结果，则说明语法是有二义性的。你可以把`parser combinators`理解为动态生成的递归下降算法的实现，不加特别处理的话是没有回溯的过程的，就只能解析出一个确定的结果（或是解析出错）。对于程序设计语言来说我觉得这不是一件坏事。

举个栗子，鄙司内部有个简单的DSL，投入使用后稳定运行了一段时间。某天某位同学突然发现自己刚写的一段DSL能把编译器卡死。调查之后我发现由于编译DSL的编译器用了个支持Earley算法[6]的库，能够解析二义性语法，所以语法规则就定义得比较放飞自我。一段特定的程序就导致编译器在不停尝试各种可能，只为得出一个解析结果。最后通过消除了语法中的二义性才解决了问题。所以我觉得并不是解析器能力越强就越好。

### 小结

这次我们讲了如何用`parser combinators`来解析表达式，你可以试试修改课上的语法，以便能够用`parser combinators`来处理。

### 参考资料

本文中解析表达式的`parser combinators`，参考了`megaparsec`[7]早期版本的API。

* [1] [06/parser.ts · RichardGong/Craft A Language - Gitee.com](https://gitee.com/richard-gong/craft-a-language/blob/master/06/parser.ts)
* [2] [03｜支持表达式：解析表达式和解析语句有什么不同？-极客时间](https://time.geekbang.org/column/article/407295)
* [3] [04 \| 语法分析（二）：解决二元表达式中的难点-极客时间](https://time.geekbang.org/column/article/120388)
* [4] [TwoHardThings](https://martinfowler.com/bliki/TwoHardThings.html)
* [5] [Exponentiation (**) - JavaScript \| MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Exponentiation#description)
* [6] [Earley parser - Wikipedia](https://en.wikipedia.org/wiki/Earley_parser)
* [7] [Text.Megaparsec.Expr](https://hackage.haskell.org/package/megaparsec-5.2.0/docs/Text-Megaparsec-Expr.html)