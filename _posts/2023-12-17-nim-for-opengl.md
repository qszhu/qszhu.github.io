---
layout: post
title: 谈谈Nim与C/C++的互操作——以OpenGL为例
date: 2023-12-17 19:00:00 +0800
tags: [nim, cpp, opengl]
---

* [谈谈Nim与JavaScript的互操作——以LeetCode为例（FFI篇）](/2023/02/18/nim-for-leetcode.html)
* **谈谈Nim与C/C++的互操作——以OpenGL为例** <-- 你在这里

Nim作为一门胶水语言, 像极了当年作为一种全栈语言大杀四方的Python. 然而对其特色的跨语言调用功能的描述却散落在不同的文档中, 缺乏有效的整理. 本文尝试整理一下Nim与C/C++互操作的用法.

在不使用3D引擎的情况下开发一个OpenGL程序可以用到许多库, 有C的也有C++的. 所以本文就以OpenGL开发为例进行整理.

## 第一个例子: GLFW[1]

首先是窗口和上下文管理. 一个最简的使用GLFW显示窗口的程序可能长这样:

```cpp
#include <GLFW/glfw3.h>

#include <iostream>

void processInput(GLFWwindow *window);

int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

    GLFWwindow* window = glfwCreateWindow(800, 600, "OpenGL", NULL, NULL);
    if (window == NULL)
    {
        std::cout << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);

    while (!glfwWindowShouldClose(window))
    {
        processInput(window);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}

void processInput(GLFWwindow *window)
{
    if(glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}
```

编译和链接代码需要给编译器传入正确的参数, 来找到头文件和库文件, 以及控制编译器的行为. Nim中通过指定pragma实现:
* `passC`[2]: 传递给编译器的参数
* `passL`[3]: 传递给链接器的参数
* `link`[4]: 静态链接

这里我们使用静态链接的方式:

```nim
# glfw.nim
{.passC: "-Ilib/glfw/include",
  link: "lib/glfw/lib/libglfw3.a".}
```

在Mac上还需要链接额外的库:

```nim
when defined(macos) or defined(macosx):
  {.passL: "-framework Cocoa",
    passL: "-framework IOKit".}
```

然后就需要通过`importc`[5]和`header`[6] pragma, 让Nim可以调用到C中的函数, 以及使用C中定义的常量等.

### 常量

C中的常量, 可以直接`importc`为Nim中的不可变量. 注意对于C中通过`define`定义的常量, Nim中对应的变量需要一个类型:

```nim
const GLFW = "GLFW/glfw3.h"

let GLFW_CONTEXT_VERSION_MAJOR* {.importc, header: GLFW.}: cint
```

枚举类型因为具体取值也是整型, 所以也可以用这种方式声明.

### 函数

常用的函数相关的pragma有:

* `cdecl`[7]: 指定使用C编译器的调用约定
* `varargs`[8]: 指定函数接受可变参数. 就算C函数不接受可变参数也可以这么写, 这样声明时就可以不用显式写出一个个参数. 但显然这样就无法提前让Nim编译器发现传参错误, 只能等到C编译器去发现.

于是要声明C中的函数可以像这样写:

```nim
proc glfwGetFramebufferSize*() {.importc, cdecl, varargs, header: GLFW.}
proc glfwGetKey*() {.importc, cdecl, varargs, header: GLFW.}: cint
```

使用`varargs`的另一个好处是`string`类型会被自动转换成`cstring`, 所以通过这种方式声明的函数在调用时可以直接传入`string`, 比如:

```nim
let colorLocation = glGetUniformLocation(shaderProgram, "color")
```

### 结构体

由于Nim中的`object`对应的就是C中的结构体, 所以C中的结构体直接`importc`到`object`就能用, 比如:

```nim
type GLFWwindow* {.importc, header: GLFW.} = object
```

### 指针

在Nim中声明带类型的指针, 可以用`ptr T`或`ptr[T]`, 两者是等价的.

由于在GLFW中, `GLFWwindow`相关函数的参数和返回值都是`GLFWwindow *`, 那么给`ptr GLFWwindow`定义个别名就能少打些字, 让Nim中的`GLFWwindow`等同于C中的`GLFWwindow *`. 像这样:

```nim
type
  GLFWwindowObj {.importc: "GLFWwindow", header: GLFW.} = object
  GLFWwindow* = ptr GLFWwindowObj
```


### 进一步简化代码

当有一大段声明都使用相同的pragma时, 可以通过`push`和`pop`[9]来简化, 比如:

```nim
{.push importc, cdecl, varargs, header: GLFW.}
proc glfwCreateWindow*(): GLFWwindow
proc glfwGetFramebufferSize*()
proc glfwGetKey*(): cint
proc glfwGetProcAddress*(): pointer
proc glfwGetTime*(): cdouble
{.pop.}
```

### 最终结果

在声明完所用到的C函数和常量之后, 上面的C程序用Nim来写就是这个样子:

```nim
import glfw

proc processInput(window: GLFWwindow)

proc main() =
  glfwInit()
  glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3)
  glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3)
  glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE)

  when defined(macos) or defined(macosx):
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GLFW_TRUE)

  let window = glfwCreateWindow(800, 600, "OpenGL", nil, nil)
  if window == nil:
    echo "Failed to create GLFW window"
    glfwTerminate()
    quit(-1)

  glfwMakeContextCurrent(window)

  while not glfwWindowShouldClose(window):
    processInput(window)

    glfwSwapBuffers(window)
    glfwPollEvents()

  glfwTerminate()

proc processInput(window: GLFWwindow) =
  if glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS:
    glfwSetWindowShouldClose(window, true)

when isMainModule:
  main()
```

可以看到代码风格类似Python脚本, 然而Nim编译器会将其转换为C代码后再编译和链接为可执行文件. 这也是Nim早期的宣传类似“Python的开发效率, C的执行速度”的由来. (然而实际用下来就会发觉Nim有着各种各样的怪癖和小毛病, 注定无法被大众所接受. 2.0版本的发布说明中甚至写道"Nim is a programming language that is good for everything, but not for everybody"[10]. 有机会再展开了.)

以上就是一个完整的Nim使用C的库的例子. 限于篇幅, 接下来的例子就按照使用场景来整理, 不再提供完整的程序了.

## 编译源文件

除了上面提到的动态链接和静态链接外, 也可以使用`compile`[11]来直接编译源文件, 例如配置完GLAD[12]后得到的源文件:

```nim
const GLAD = "glad/glad.h"
{.passC: "-Ilib/glad/include",
  compile: "lib/glad/src/glad.c".}
```

## 指针

C中的函数要改变传参的取值的话, 参数需要是指针类型. 例如:

```c
int success;
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
```

Nim中的`addr`对应了取址操作符`&`, 所以可以这样写:

```nim
var success: cint
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, addr(success));
```

### 字符串指针

上面提到使用`varargs`声明的函数, 传入的`string`会被自动转换成`cstring`. 但在需要修改`string`的内容时编译器会警告. 比如这样的情况:

```c
char infoLog[512];
glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
```

这时就需要用显示类型转换:

```nim
let infoLog = newString(512).cstring
glGetShaderInfoLog(vertexShader, 512, nil, infoLog)
```
### 函数指针

GLAD中有需要传入函数指针的情况, 比如:

```c
if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) {
```

Nim中的`pointer`就相当于`void *`, 所以可以这样声明:

```nim
type GLADloadproc* {.importc, header: GLAD.} = pointer

proc gladLoadGLLoader*() {.importc, cdecl, varargs, header: GLAD.}: cint
```

这样上面的C代码对应的Nim代码就可以写成:

```nim
if gladLoadGLLoader(cast[GLADloadproc](glfwGetProcAddress)) == 0:
```

## 数组

C中的数组和指向数组第一个元素的指针是一样的, 所以需要传数组的时候也可以传指针. 比如:

```c
unsigned int VAO;
glGenVertexArrays(1, &VAO);
```

可以写成:

```nim
var VAO: uint32
glGenVertexArrays(1, addr(VAO))
```

当然直接传数组也可以, 甚至不需要取址:

```nim
var VAOs: array[2, uint32]
glGenVertexArrays(2, VAOs)
```

如果是传`seq`类型, 那么可以传第一个元素的地址:

```nim
let vertices = @[
  #...
].mapIt it.float32
glBufferData(GL_ARRAY_BUFFER, vertices.len * sizeof(cfloat), addr(vertices[0]), GL_STATIC_DRAW)
```

## 类型大小

Nim中的`sizeof`和C中的用法一样, 只是类型必须是对应的C类型. 比如:

```c
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
```

就可以写成:

```nim
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(cfloat), cast[pointer](0))
```

## C++

由于C和C++是不同的语言, 所以对应的例子也需要分开讲. 首要的不同在于编译时的参数. 编译调用C库的Nim程序时使用`nim c`, C++时使用`nim cpp`. 此外, `importc`要改为`importcpp`[13].

### 命名空间

命名空间[14]需要在`importcpp`时指定, 比如GLM[15]中的类型:

```nim
type Vec3* {.importcpp: "glm::vec3", header: GLM.} = object
  x*, y*, z*: float
```

### 函数

即便在`importcpp`时可以使用`@`来表示所有剩余的参数, C++函数在声明时也需要写出所有参数. 比如:

```nim
proc radians*(deg: cfloat): cfloat {.importcpp: "glm::radians(@)", header: GLM.}
```

#### 构造函数

构造函数因为调用时的特殊形式, 需要使用`constructor`[16] pragma. 比如:

```nim
proc initVec3*(x, y, z: cfloat): Vec3 {.importcpp: "glm::vec3(@)", constructor, header: GLM.}
```

### 运算符重载

Nim中也有运算符重载, 配合`importcpp`可以调用到C++中对应的版本. 比如:

```nim
proc `+`*(a, b: Vec3): Vec3 {.importcpp: "# + #".}
```

### 常量指针作为函数返回值

由于Nim中没有常量指针, 所以遇到返回值为常量指针的函数, 不得已只能转换为非常量指针. 比如ASSIMP[17]中的`const char * Assimp::Importer::GetErrorString() const`:

```nim
proc GetErrorString*(self: Importer): cstring
  {.importcpp: "(char *)#.GetErrorString()", header: ASSIMP_IMPORTER.}
```

### 继承

Nim 2.0引入了`virtual`[18] pragma, 使得继承C++中的类以及覆盖类方法成为可能. 例如使用openFrameworks[19]时需要继承`ofBaseApp`并覆盖其中相应的方法:

```cpp
class ofApp : public ofBaseApp {
public:
  void setup();
  void update();
  void draw();
}
```

Nim中就可以这样实现:

```nim
type
  ofApp = object of ofBaseApp

proc newOfApp(): ofApp {.constructor: "ofApp(): ofBaseApp()".} =
  discard

proc setup(self: ofApp) {.virtual.} =
  discard

proc update(self: ofApp) {.virtual.} =
  discard

proc draw(self: ofApp) {.virtual.} =
  discard
```

## 小结

以上整理的情况应该能覆盖大部分与C/C++互操作的情况. 对于小规模的调用, 这样手动声明绑定的方式就够用了, 且不会引入其他依赖. 但对于大规模的库调用, 最好能找到现成的绑定或是自动生成绑定. 比如OpenGL的场合就可以使用nimgl[20].

## 参考

* [1] [GLFW](https://www.glfw.org/)
* [2] [passc pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-passc-pragma)
* [3] [passl pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-passl-pragma)
* [4] [Link pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-link-pragma)
* [5] [Importc pragma - Nim Manual](https://nim-lang.org/docs/manual.html#foreign-function-interface-importc-pragma)
* [6] [Header pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-header-pragma)
* [7] [Procedural type - Nim Manual](https://nim-lang.org/docs/manual.html#types-procedural-type)
* [8] [Varargs pragma - Nim Manual](https://nim-lang.org/docs/manual.html#foreign-function-interface-varargs-pragma)
* [9] [push and pop pragmas - Nim Manual](https://nim-lang.org/docs/manual.html#pragmas-push-and-pop-pragmas)
* [10] [Nim v2.0 released - Nim Blog](https://nim-lang.org/blog/2023/08/01/nim-v20-released.html)
* [11] [compile pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-compile-pragma)
* [12] [Glad](https://glad.dav1d.de/)
* [13] [Importcpp pragma - Nim Manual](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-importcpp-pragma)
* [14] [Namespaces - Nim Manual](https://nim-lang.org/docs/manual.html#importcpp-pragma-namespaces)
* [15] [OpenGL Mathematics](https://glm.g-truc.net/0.9.9/index.html)
* [16] [Wrapping constructors - Nim Manual](https://nim-lang.org/docs/manual.html#importcpp-pragma-wrapping-constructors)
* [17] [The Asset Importer Library](https://assimp.org/)
* [18] [C++ interop enhancements - Nim v2.0 released - Nim Blog](https://nim-lang.org/blog/2023/08/01/nim-v20-released.html)
* [19] [openFrameworks](https://openframeworks.cc/)
* [20] [Github - nimgl/nimgl](https://github.com/nimgl/nimgl)
