---
layout: post
title: 通过WASM在编程竞赛平台使用Nim
date: 2023-12-18 19:00:00 +0800
tags: [nim, oj, wasm]
---

Nim作为一门小众且相对年轻的语言, 显然不会被大部分编程竞赛平台采纳(AtCoder是个例外). 不过得益于其互操作能力, 使得在某些平台上可以“曲线救国”. 比如之前尝试过的[LeetCode](/2023/02/18/nim-for-leetcode.html), 就是通过编译到JavaScript的方式实现的.

这次我们换个思路. 既然Nim可以[编译到C/C++](/2023/12/17/nim-for-opengl.html), 那么是否可以提交编译后的C/C++代码呢? 实际试下来即便可行也会非常繁琐, 原因大致就是编译出来的C/C++代码依赖于Nim和系统的各种头文件, 难以得到可独立提交且文件大小被各平台接受的单一文件.

于是我们再次换个思路, 通过emscripten[1]将C/C++代码编译成WASM模块, 再通过平台支持的Node.js来运行. 实测下来这个思路是可行的, 然而相关的信息也是散落在不同的文档里. 在此记录一下.

## JavaScript胶水程序

简单起见, 约定WASM模块需要提供一个函数solve(), 参数是平台运行程序时的标准输入, 返回值为要写入标准输出的字符串. 这样, 通过Node.js运行的JavaScript程序在加载了WASM模块后, 就可以直接输出调用solve()函数的返回值.

这样一个JavaScript胶水程序长这样:

```javascript
const chunks = []

process.stdin.on('data', chunk => chunks.push(chunk))
process.stdin.on('end', () => {
  let lines = Buffer.concat(chunks) // 1)

  let inst = Module() // 2)
  let buf = inst._malloc(lines.length) // 3)
  inst.HEAPU8.set(lines, buf) // 4)

  let resBuf = inst.ccall('solve', 'number', ['number'], [buf]) // 5)
  let output = inst.UTF8ToString(resBuf) // 6)

  console.log(output)
})
```

1) 标准输入的内容被存储到lines变量中

2) 初始化WASM模块

3) 在WASM模块中分配一块内存用来存储标准输入

4) 将标准输入的内容复制到WASM模块中刚申请的内存中

5) 调用WASM模块中的solve方法, 得到存储着返回值的内存块的指针

6) 将返回的内存块转换成UTF8字符串

更详细的解释可以参考emscripten的文档[2].

## 编译器配置文件

在使用emscripten将C/C++代码编译为WASM时, 大部分情况下只需要把gcc/clang替换成emcc就行. 这里就需要告诉Nim编译器在编译和链接时使用emcc. 由于还需要给Nim编译器传入很多参数, 可以把这些参数都写在一个`config.nims`[3]文件里, Nim编译器在运行时就会自动读取这些参数:

```nim
--cc:clang
--clang.exe:emcc
--clang.linkerexe:emcc
```

因为编译WASM也是一种交叉编译[4], 还需要指定cpu为wasm32:

```nim
--cpu:wasm32
```

接下来需要给emcc传入一些参数. 因为是链接阶段的参数, 所以要用`passL`让Nim编译器来传达.

首先是导出在JavaScript文件里需要用到的函数[6]:

```nim
switch "passL", "-sEXPORTED_FUNCTIONS=_solve,_malloc"
switch "passL", "-sEXPORTED_RUNTIME_METHODS=ccall,UTF8ToString"
```

将WASM模块嵌入到JavaScript文件中[7]:

```nim
switch "passL", "-sSINGLE_FILE"
```

JavaScript胶水文件(假设命名为`post.js`)需要被拼接到生成的文件最后[8]:

```nim
switch "passL", "--extern-post-js post.js"
```

为了能像在上述胶水文件中同步初始化WASM模块, 需要加入如下参数[9][10]:

```nim
switch "passL", "-sWASM_ASYNC_COMPILATION=0"
switch "passL", "-sMODULARIZE"
```

## 目标文件瘦身

使用上述参数能够编译出独立的.js文件, 然而文件大小通常会有数百KB, 远超一般平台允许上传的最大文件大小. 于是还需要传入些能减小目标文件大小的参数.

关闭Nim的运行时检查可以去掉这些运行时检查的代码[11]:

```nim
--define:danger
```

因为只会用到solve()函数, 所以不需要生成main()函数[5]:

```nim
switch "noMain", "on"
```

告诉emcc优化目标文件大小[12]:

```nim
switch "passL", "-Os"
```

这些额外的操作能把目标文件缩减到几十KB. 然而运行时会抛出异常, 类似:

```
CompileError: WebAssembly.Module(): Compiling function #7 failed: Invalid opcode (enable with --experimental-wasm-threads) @+547
```

看上去是由于WASM尚不支持线程导致, 于是告诉Nim不要使用多线程[5]:

```nim
--threads:off
```

## 内存限制

上述方案有个限制, 就是必须将标准输入全部复制到WASM模块中. 这就要求WASM模块能够自主扩展内存[13]:

```nim
switch "passL", "-sALLOW_MEMORY_GROWTH"
```

另一个限制是堆栈大小. 这会在需要用递归方法解题时直接影响递归的深度:

```nim
switch "passL", "-sSTACK_SIZE=128MB"
```

此时Node.js也需要传入V8的参数`"--stack_size"`[14].

## 具体平台的问题

### AtCoder

虽然AtCoder官方支持Nim, 不过版本长期停留在1.0.6, 标准库都缺少很多常用方法, 不是很好用. 而经过上述以Node.js运行WASM模块的方式提交的话, 可以用上最新的2.0版本. 不过有些依赖原生库的标准库(如`std/re`)还是会报错.

此外, AtCoder中Node.js的启动参数有误[15], 导致堆栈大小一直是默认值:

```bash
node {dirname}/{basename} --stack-size={stack_size:kb}
```

这个问题直到最近才得到修正[16]:

```bash
node --stack-size={memory:kb} Main.js ONLINE_JUDGE ATCODER
```

### CodeForces

CodeForces对于内存卡得比较严格, 所以上述方法在遇到大规模的输入或是大的堆栈深度就会报错.

### LeetCode

无论如何调整编译器参数, 在LeetCode上运行WASM都会报内存不足, 一度把我给整不会了. 好在LeetCode上可以自由运行一些系统命令, 让我们可以看到运行环境的配置:

```javascript
const cp = require('child_process')
console.log(String(cp.execSync("ulimit -v")))
```

上述命令返回的结果是`1171875`, 意味着运行环境限制的虚拟内存大小为1.1G左右. 而可能鲜为人知的是, V8运行WASM所需的虚拟内存为10G[17]. 虽然V8分配虚拟内存的方式并不导致实际的内存占用, 但在限制虚拟内存的环境中就无法运行WASM了.

## 小结

在编程竞赛平台上提交程序并运行的行为, 从广义上看其实也是一种部署行为. 需要在受限的环境中进行部署的情况, 在日常工作中也并不少见. 从这个角度看的话, 编程竞赛也不仅仅是考验算法了.

把WASM看作一个新的部署对象的话, 经由LLVM-Emscripten就能够让一众语言都能通过这种方式运行在允许WASM运行的地方. 按照维基[18]上的记述, 这样的语言包括且不限于Ada, C, C++, D, Delphi, Fortran, Haskell, Julia, Objective-C, Rust和Swift等. 感兴趣的同学可以试试通过这种方式在不提供官方支持的编程平台上提交自己喜欢的语言.

## 参考

* [1] [Emscripten](https://emscripten.org/index.html)
* [2] [Interacting with code - Access memory from Javascript - Emscripten documentation](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#access-memory-from-javascript)
* [3] [NimScript](https://nim-lang.org/docs/nims.html)
* [4] [Cross-compilation - Nim Compiler User Guide](https://nim-lang.org/docs/nimc.html#crossminuscompilation)
* [5] [Command-line switches - Nim Compiler User Guide](https://nim-lang.org/docs/nimc.html#compiler-usage-commandminusline-switches)
* [6] [Interacting with code - Calling compiled C functions from JavaScript using ccall/cwrap - Emscripten documentation](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#calling-compiled-c-functions-from-javascript-using-ccall-cwrap)
* [7] [Emscripten linker ouput files - Emscripten documentation](https://emscripten.org/docs/compiling/Building-Projects.html#emscripten-linker-output-files)
* [8] [Emscripten Compiler Frontend (emcc) - Arguments - Emscripten documentation](https://emscripten.org/docs/tools_reference/emcc.html#arguments)
* [9] [FAQ - How can I tell when the page is fully loaded and it is safe to call compiled function? - Emscripten documentation](https://emscripten.org/docs/getting_started/FAQ.html#how-can-i-tell-when-the-page-is-fully-loaded-and-it-is-safe-to-call-compiled-functions)
* [10] [Building to WebAssembly - .wasm files and compilation - Emscripten documentation](https://emscripten.org/docs/compiling/WebAssembly.html#wasm-files-and-compilation)
* [11] [Additional compilation switches - Nim Compiler User Guide](https://nim-lang.org/docs/nimc.html#additional-compilation-switches)
* [12] [Optimizing Code - How to optimize code - Emscripten documentation](https://emscripten.org/docs/optimizing/Optimizing-Code.html#how-to-optimize-code)
* [13] [Optimizing Code - Memory Growth - Emscripten documentation](https://emscripten.org/docs/optimizing/Optimizing-Code.html#memory-growth)
* [14] [FAQ - Why do I get a stack size error when optimizing? - Emscripten documentation](https://emscripten.org/docs/getting_started/FAQ.html#why-do-i-get-a-stack-size-error-when-optimizing-rangeerror-maximum-call-stack-size-exceeded-or-similar)
* [15] [Language Test 202001 - AtCoder](https://atcoder.jp/contests/language-test-202001)
* [16] [Language Test 202301 - AtCoder](https://atcoder.jp/contests/language-test-202301)
* [17] [Chasing Memory Bugs through V8 and WebAssembly](https://blog.stackblitz.com/posts/debugging-v8-webassembly/#but-why%3F)
* [18] [LLVM - Frontends](https://en.wikipedia.org/wiki/LLVM#Frontends)