---
layout: post
title: LeetCode TypeScript解题工具
date: 2020-08-15 14:41:00 +0800
tags: [typescript, leetcode]
---

### 1. 背景

最近从某个渠道获赠了`LeetCode`会员，心想不用白不用，就上去看了一下。结果十分震惊。一是有了独立的中文网站，还有定期比赛每日打卡等，看上去似乎搞得有声有色。二是如今竟然已经有了近2000题，俺当年刷完的100多题顿时显得微不足道。三是支持的语言多了许多，`Go`、`Swift`、`Rust`等如今迈入主流的语言都有了。

不过其中`TypeScript`的支持还标着"Beta"字样。其实只要网站支持`JavaScript`，就可以自行转译`TypeScript`代码为`JavaScript`代码后提交。这也是工业界在部署`TypeScript`工程时的做法。相比`LeetCode`上的“开箱即用”，缺点是需要在本地先搭建工程框架，优点则是可以方便引入第三方库，以及可以自定义编译器选项和代码检查规则等。

结果就是[ts-leetcode](https://github.com/qszhu/ts-leetcode)项目，提供了在本地编辑ts题解和测试提交的功能。虽然功能说起来简单，但项目到达当前的状态，也先后经历了三个版本，值得说一下。

### 2. V1

第一个版本[1]用`Hygen`[2]生成本地的解题模版[3]。特点是可在本地运行单元测试，并且用`nodemon`对更新的代码持续进行测试，比较符合业界的通用做法。对于生成的代码还可以选择进行压缩和混淆。

第一版虽然已经可用，但缺点也很明显：

* 首先是模版的适用范围不足。比如`LeetCode`上不仅仅有算法题，还有设计题等，模版就应该是不同的，但在不读取题目内容的情况下就是无法区分的。
* 其次是测试数据的形式多样。比如链表二叉树和图等，需要在测试代码中把测试数据反序列化成规定的数据结构。而这部分工作`LeetCode`网站已经做了，没必要重复做。

以上这些都说明了需要与`LeetCode`网站进行更深入的交互，于是就有了第二个版本。

### 3. V2

第二个版本[4]的核心是与`LeetCode`网站的交互，所以需要捕获浏览器向网站发送的请求后进行分析。用到的接口有这么几个：

#### 获取题目信息（`POST /graphql/`）

可以看到这是一个`graphql`接口（查询`questionData`）所以我们可以自行选择需要返回的数据：

* `sampleTestCase`: 题目里提供的样例测试数据
* `metaData`: 要实现的函数或是类的签名，用于从模版生成解题代码

#### 测试代码（`POST /problems/:titleSlug/interpret_solution/`）

用于自行比对测试数据的接口，上传代码后会返回：

* `interpret_id`: 上传代码的运行任务id
* `interpret_expected_id`: 标准程序的运行任务id

需使用下面的结果检查接口获得运行结果

#### 检查运行结果（`GET /submissions/detail/:id/check/`）

传入执行任务的id即可知道运行状态和输出结果等

#### 提交代码（`POST /problems/:titleSlug/submit/`）

上传代码后会返回`submission_id`，再使用结果检查的接口检查即可

#### 用户认证

除了获取题目信息的接口以外，上述的接口都需要进行用户认证。请求cookie中需带上`LEETCODE_SESSION`和`csrfToken`。不过实际测试时发现依旧会返回CSRF验证错误的信息。更多测试之后发现起作用的是`Referer`头，甚至不带`csrfToken`也可以（感觉可以搞点事情……）

至此整个项目自用已经差不多了，但支持用的脚本和解题工程文件都混在一起，不利于发布。于是又有了第三个版本。

### 4. V3

第三个版本是当前版本[5]，主要是把命令行工具单独拆出来，方便发布。此外还用`webpack`进行打包，这样就可以引入第三方库使用了。

其实第一版的时候尝试过使用`webpack`，不过当时想要搞成多个`entry`和多个`output`，结果没搞起来。其实`webpack`的config可以返回一个函数[6]，这样就可以传入不同的参数来改变`entry`和`output`了。

此外`output`需要输出一个命名的library[7]，因为`LeetCode`似乎是在你上传的代码后直接拼接其他的支持代码再运行的，所以需要能够在当前文件中找到题目规定的函数名或类名。

### 5. 其他

实际使用的话，设置`LEETCODE_SESSION`这一步比较麻烦，需要用户具备一些技术能力。而且cookie过期后还要重新设置。更直观的做法应该是支持自动登录的，不过看了下似乎有点复杂。`leetcode-cn.com`的登录整合了阿里云的无痕验证[8]，登录时会发送一串浏览器本地生成的数据，没有的话就直接拒绝请求了，等有时间再搞了。看了下相关功能介绍，也无怪乎一个刷题的网站会发送阿里系的cookie了。将来是不是会这边刚刷完题，那边淘宝就推送键盘鼠标显示器的链接呢？🤪

### 参考资料
* [1] [GitHub - qszhu/ts-leetcode at v1](https://github.com/qszhu/ts-leetcode/tree/v1)
* [2] [Hygen \| Hygen](https://www.hygen.io/)
* [3] [ts-leetcode/_templates/leetcode/with-prompt at v1 · qszhu/ts-leetcode · GitHub](https://github.com/qszhu/ts-leetcode/tree/v1/_templates/leetcode/with-prompt)
* [4] [GitHub - qszhu/ts-leetcode at v2](https://github.com/qszhu/ts-leetcode/tree/v2)
* [5] [GitHub - qszhu/ts-leetcode](https://github.com/qszhu/ts-leetcode)
* [6] [Configuration Types \| webpack](https://webpack.js.org/configuration/configuration-types/#exporting-a-function)
* [7] [Output \| webpack](https://webpack.js.org/configuration/output/#outputlibrary)
* [8] [功能概述_无痕验证_人机验证-阿里云](https://help.aliyun.com/document_detail/122071.html)
