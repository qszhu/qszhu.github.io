---
layout: post
title: JavaScript文学编程 
date: 2020-07-26 03:02:00 +0800
tags: [javascript]
---

副标题：在`JavaScript`生态中获得类似`Jupyter Notebook`的体验

### 1. 背景

作为软件开发人员，发布一个模块之前需要准备好相应的文档（这里特指API文档）。传统的API文档(生成器)有一些缺陷：

* 如果只提供接口的描述，而不提供具体的示例代码的话，使用者第一次写出来的接口调用代码很大概率是无法使用的；
* 如果只提供了调用的示例而不提供代码运行的结果的话，使用者第一次写出来的结果处理代码很大概率是无法使用的；
* 如果需要同时提供示例的代码和运行的结果，则需要手工运行一段段代码，复制粘贴运行结果。于是除了需要保持文档与接口的同步更新之外，还要保持文档中示例代码和运行结果的更新。程序员不想写文档的理由++。

业界很早就注意到了这点，所以现在有些Web API文档生成器甚至提供了在线调用沙盒接口的能力，能让使用者直接运行文档中的示例代码。

`Python`生态在这方面就做得很好。比如有`doctest`[1]，在代码的注释中就提供了文档和示例，并且是可以执行和比对执行结果的：

```python
"""
This is the "example" module.

The example module supplies one function, factorial().  For example,

>>> factorial(5)
120
"""
```

更不用提`Jupyter Notebook`[2]了，基本是机器学习领域的标配了。其本身就是借鉴了文学编程的思想。文档和代码可以交织在一起，并且代码都可以直接执行。

相比之下，`JavaScript`生态中就缺少类似的大杀器。早年在`CoffeeScript`中是部分支持的[3]，但在`ES6`和`TypeScript`出来后新工程用`CoffeScript`的应该很少了。

`Jupyter Notebook`在设计上支持不同语言的kernel，搜了一下确实存在js/ts的kernel，不过使用之前先要装一堆`Python`的依赖。有没有纯js的方案呢？找了一圈似乎是没有，于是就自己实现下试试看。

### 2. 实现

实现拢共分两步：

1. 从文档中提取出代码片段
2. 运行代码片段，把运行结果写回到文档中

第一步很简单。因为文档采用`Mardown`格式，只需把其中标注的代码块提取出来，甚至都不需要专门的`Mardown`解析器。简单起见，这里就提取 \`\`\`javascript 和 \`\`\` 之间的内容。

第二步则是通过`Node.js`的`vm`模块[4]实现。通过`vm.createContext()`方法可以为要执行的代码创建一个沙盒环境，通过`vm.runInContent()`方法在沙盒环境中执行代码。代码中创建的变量会留在沙盒环境中，并且代码最后一句语句的执行结果会作为返回值返回给执行者。

输出结果是这个样子的：

### 字符串

```javascript
const hello = 'hello'
hello
```
Output:
```javascript
'hello'
```


### 数字

```javascript
const a = 42
a
```
Output:
```javascript
42
```


### 对象

```javascript
const obj = {
  foo: 'bar',
  123: 456
}
obj
```
Output:
```javascript
{ '123': 456, foo: 'bar' }
```


### 跨block

```javascript
const world = hello + ' world'
world
```
Output:
```javascript
'hello world'
```

### 函数

```javascript
function add(a, b) {
  return a + b
}
add(1, 2)
```
Output:
```javascript
3
```

### 3. async函数

ES6支持async函数，用这样的方法就行不通。因为async函数的结果可能不会在当前的事件循环中返回。这里就利用沙盒环境hack了一下，让调用方等待async函数resolve：

```javascript
async function runAsyncBlock(code) {
  await new Promise((resolve, reject) => {
    sandbox._resolve = resolve
    sandbox._res = undefined
    sandbox._err = undefined
    sandbox._lastRes = undefined
    const wrapped = `
      (async () => {
        ${code}
      })()
        .then(r => {
          _res = r
          _resolve()
        })
        .catch(e => {
          _err = e
          _resolve()
        })
    `
    const script = new vm.Script(wrapped)
    const context = vm.createContext(sandbox)
    script.runInContext(context)
  })
  sandbox._lastRes = sandbox._err || sandbox._res
  return inspect(sandbox._lastRes)
}
```

为了与普通的代码作区分，这里仿照`Jupyter Notebook`里定义了一个magic指令，结果是这样的：

```javascript
%%async
async function egg() {
  return Promise.resolve('spam')
}
const spam = await egg()
return spam
```
Output:
```javascript
'spam'
```

因为hack的关系，这里还需要显式地`return`结果才行。

### 4. 模块

因为沙盒环境不带`require`，所以需要从调用方传入：

```javascript
sandbox.require = require
```

结果是这样的：

### 标准库
```javascript
const fs = require('fs')
const data = fs.readFileSync('./test.md')
data.toString().substring(0, 100)
```
Output:
```javascript
'# 测试\n\n## 变量\n\n### 字符串\n\n```javascript\nconst hello = \'hello\'\nhello\n```\n\n### 数字\n\n```javascript\nconst a ='
```


### 第三方库
```javascript
%%async
const axios = require('axios')
const res = await axios.get('https://www.163.com')
return res.data.length
```
Output:
```javascript
496870
```

```javascript
const mysql = require('mysql')
mysql
```
Output:
```javascript
{ createConnection: [Function: createConnection],
  createPool: [Function: createPool],
  createPoolCluster: [Function: createPoolCluster],
  createQuery: [Function: createQuery],
  escape: [Function: escape],
  escapeId: [Function: escapeId],
  format: [Function: format],
  raw: [Function: raw] }
```

但这样使用有个前提，就是必须得在目标工程的目录下运行才能找得到第三方依赖。这就使得这个工具没法独立发布了。

### 参考资料
* [1] https://docs.python.org/2/library/doctest.html
* [2] https://jupyter.org/
* [3] http://coffeescript.org/#literate
* [4] https://nodejs.org/api/vm.html
