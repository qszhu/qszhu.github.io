---
layout: post
title: 迁移现有Koa项目到腾讯serverless服务
date: 2020-06-04 18:02:00 +0800
tags: [tencent_cloud, serverless]
---

提示: 腾讯云将在存量组件全部更新到v2后默认展示v2的文档，YMMV。

### 1. 背景

[上一篇](/2020/05/23/tencent-scf.html)提到，腾讯云云函数对于函数的入口有要求，这可能会影响项目迁移，造成提供商绑定。对此serverless的解决方案是components[1]，而腾讯云也已经支持了。

上一篇中用到的网页转图片服务，其实原本是一个Koa工程，在腾讯云的serverless framework产品页[2]上可以找到对应的组件`tencent-koa`[3]。于是我们就来看一下迁移一个现有的项目会有多简单（或是有多难）。

### 2. 迁移问题

#### 2.1 模块导出

从`tencent-koa`的README中可以看到，该组件需要一个`app.js`，在其中通过`module.exports`导出Koa的app实例：

```javascript
// app.js
// ...
module.exports = app
```

我们现有的工程已经有了一个`app.ts`，并导出了app实例，不过实际使用时却发生了错误。我们的`app.ts`是这样的：

```typescript
// app.ts
// ...
export default app
```

经过`tsc`编译以后会输出：

```typescript
// ...
exports.default = app;
```

这是es模块的default导出，而`tencent-koa`需要的是一个commonjs模块。出错是因为两者不兼容。解决的办法有这么几种：

* 通过`babel`再把es模块转换为commonjs模块。我们的项目以前就是用`babel`来将typescript转译为javascript的，但现在已经去掉了`babel`来直接用`typescript`的工具转译了，不大想再走回头路。
* 至于`typescript`为什么不像`babel`一样兼容commonjs模块，据说typescript团队内部也进行过讨论[4]，结论就是只支持纯es模块。不过typescript提供了另一种写法，写成`export = app`的话就会输出`module.exports = app`。不过这样还是需要修改已有的工程文件。

最后采用的方案是fork了`tencent-koa`项目，修改其中导入模块的方式，使其兼容es模块的default导出。这样就不用修改已有的工程文件了🐶

##### 2.1.1 Component开发

在`serverless.yml`中，`component`属性可以支持本地（绝对）路径，所以将其改成fork后的`tencent-koa`代码目录就可以进行本地的Component开发了。

通过查看源码发现，模块导入功能的实现是由`tencent-koa`项目通过`@ygkit/bundler`[5]项目，借由`webmake`[6]项目实现的（题外话：找到这层关系还花了些功夫，详见[这篇](/2020/05/27/tencent-serverless.html)）。实际调用的就是一个`require`。这里仿照`babel`，在`require`之后调用如下的helper函数：

```javascript
function _interopRequireDefault(obj) {
  return obj && obj.__esModule ? obj : { default: obj }
}
```

这样就可以通过`default`属性来访问没有default导出的commonjs模块了。

实际开发中还遇到的问题是`tencent-koa`会把打包生成的代码放在`~/.serverless/cache/`目录中，所以每次改动代码都需要清除这个cache，否则改动就无法生效。

至此就实现了在不改动已有工程代码的情况下部署到腾讯云。

#### 2.2 debug日志

我们已有的工程是通过`debug`[7]来输出日志的。不幸的是，在腾讯云云函数的控制台看不到这些日志的输出。通过查看源码发现`debug`默认使用`process.stderr`进行输出。开了腾讯云的工单确认腾讯云目前不支持输出到`process.stdout`和`process.stderr`。而根据`debug`的文档[8]，通过

```javascript
debug.log = console.log.bind(console)
```

让`debug`使用`console.log`输出也不行，这就有些奇怪了。通过`console.log(console)`可以在腾讯云的控制台看到如下输出：

```javascript
Console {
  log: [Function: prettyConsoleLog],
  info: [Function: prettyConsoleLog],
  warn: [Function: prettyConsoleLog],
  error: [Function: prettyConsoleLog],
  dir: [Function: bound consoleCall],
  time: [Function: bound consoleCall],
  timeEnd: [Function: bound consoleCall],
  trace: [Function: bound consoleCall],
  assert: [Function: bound consoleCall],
  clear: [Function: bound consoleCall],
  count: [Function: bound consoleCall],
  countReset: [Function: bound countReset],
  group: [Function: bound consoleCall],
  groupCollapsed: [Function: bound consoleCall],
  groupEnd: [Function: bound consoleCall],
  Console: [Function: Console],
  debug: [Function: debug],
  dirxml: [Function: dirxml],
  table: [Function: table],
  markTimeline: [Function: markTimeline],
  profile: [Function: profile],
  profileEnd: [Function: profileEnd],
  timeline: [Function: timeline],
  timelineEnd: [Function: timelineEnd],
  timeStamp: [Function: timeStamp],
  context: [Function: context],
  [Symbol(counts)]: Map {} }
```

可以看到诸如`console.log`、`console.error`等方法会去调用名为`prettyConsoleLog`的方法，猜测是腾讯云对V8作了改动，看上去是没法指望了。

对剩下的方法作了一遍测试，发现只有以下`console`方法的输出可以在控制台中看到：

```
console.time, console.timeEnd
console.trace
console.assert
console.count
console.group, console.groupCollapsed
```

不过它们或多或少都有些额外输出，唯独`console.group`，照理输出会缩进，但在控制台中看不到缩进。于是我们就有了个很hack的方法🐶：

```javascript
debug.log = console.group.bind(console)
```

通过注入这样一个修改过的`debug`实例，我们原有工程中的日志输出都可以在腾讯云的控制台中看到了。

#### 2.3 CORS

我们原有的项目自带CORS支持，但似乎云函数并不会把CORS相关的http头返回。这有其合理性。因为云函数被设计成处理单次无状态的http请求，而一些情况下浏览器会在CORS请求前先发送`OPTIONS`请求。

在腾讯云上可以在API网关中设置启用CORS。不过似乎`tencent-koa`的api网关配置中并没有提供CORS的配置。但在`tencent-apigateway`组件[9]中是提供了这个选项的。于是我们又fork了🐶。这次fork的项目是`tencent-framework`[10]。修改后[11]就可以在项目的`serverless.yml`中打开CORS选项了。

#### 2.4 环境变量

我们原有的项目通过`.env`文件在部署时改变应用的行为。在`serverless.yml`中也可以通过`environment`属性来设置环境变量。不过这样可能有个问题。因为有些私密信息如访问密钥等也是写在`.env`文件里的，所以一般是不把`.env`文件加入版本控制的。而`serverless.yml`通常是要加入版本控制的，在其中配置密钥的环境变量的话就会造成信息泄漏。好在serverless允许引用`.env`中的变量[12]。然而在写成了`XXX: ${env:XXX}`之后，serverless居然报错了：

```bash
Error: invalid reference ${env:XXX}
```

报错的代码来自一个叫`@serverless/template`的包[13]，似乎没有在github上开源。在本地安装的代码中可以看到是这样写的：

```javascript
        if (/\${env\.(\w*:?[\w\d.-]+)}/g.test(match)) {
          newValue = process.env[referencedPropertyPath[1]]
          variableResolved = true
        } else {
          if (!template[referencedTopLevelProperty]) {
            throw Error(`invalid reference ${match}`)
          }
```

等等，所以应该是写成`${env.XXX}`而不是`${env:XXX}`吗？难道是文档错了吗？搜了下发现有人有相同的疑问[14]，官方的回复是`${env.XXX}`是components v1的写法[15]，而`${env:XXX}`是v2的写法。

敢情我看了半天代码和文档，看的都是v1吗🤦‍♂️

真相是github上`serverless`文档已经默认是v2版本了，而腾讯云的组件默认还是v1的，虽然也有v2分支……

### 3. Component V2

于是我们来看一下v2是否解决了我们上面遇到的问题。

#### 3.1 模块导出

`tencent-koa` v2要求导出app实例的模块名为`sls.js`，这样其实就可以通过新增文件的方式，在不改变原有工程文件的情况下改变模块的导出方式了：

```typescript
// sls.ts
import app from './app'

export = app
```

#### 3.2 debug日志

这个是腾讯云的限制，还是通过上面hack的方式解决。

#### 3.3 CORS

`tencent-koa` v2已经支持在`serverless.yml`中启用api网关的CORS设置了[16]。

#### 3.4 环境变量

采用`${env:XXX}`的写法，可以正常运行。不过需要手动把`.env`里的配置搬到`serverless.yml`中。可能的改进方式是自动在yml中生成，或是能够像`docker-compose.yml`一样直接引用外部的`.env`文件。

(update 2010-06-29) 写了个自动把`.env`中的配置搬到`serverless.yml`中的[脚本](https://github.com/qszhu/slsenv)

### 4. 小结

可以看到如果一开始就从v2开始，就不会有那两次fork了。可能腾讯云的component v2还没准备好，代码就没有合入主线。不过最新的腾讯云的文档示例中已经在用v2了[17]，大概只能怪我看的时机不巧了🐶

另外，serverless官网对component的描述是：

> Serverless Components are designed to be entirely vendor agnostic, enabling you to easily use services from different vendors, together! 

所以是不是能够想象一下，比如腾讯云云函数不支持debug日志，我就能用另一家提供支持的云服务商的云函数组件来替换呢？这个就等有时间再试验了。

### 参考资料
* [1] [Serverless Components](https://www.serverless.com/components/)
* [2] [Serverless Framework 产品概述 - 产品简介 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/1154/38787)
* [3] [GitHub - serverless-components/tencent-koa: Easily deploy serverless Koa applications to Tencent Cloud with the Serverless Framework](https://github.com/serverless-components/tencent-koa)
* [4] [ES6 Modules default exports interop with CommonJS · Issue #2719 · microsoft/TypeScript · GitHub](https://github.com/microsoft/TypeScript/issues/2719) 
* [5] [ygkit/packages/bundler at master · yugasun/ygkit · GitHub](https://github.com/yugasun/ygkit/tree/master/packages/bundler)
* [6] [GitHub - medikoo/modules-webmake: Bundle CommonJS/Node.js modules for web browser](https://github.com/medikoo/modules-webmake)
* [7] [GitHub - visionmedia/debug: A tiny JavaScript debugging utility modelled after Node.js core’s debugging technique. Works in Node.js and web browsers](https://github.com/visionmedia/debug)
* [8] [GitHub - visionmedia/debug: A tiny JavaScript debugging utility modelled after Node.js core’s debugging technique. Works in Node.js and web browsers](https://github.com/visionmedia/debug#output-streams)
* [9] [tencent-apigateway/configure.md at master · serverless-components/tencent-apigateway · GitHub](https://github.com/serverless-components/tencent-apigateway/blob/master/docs/configure.md#endpoints-param-description)
* [10] [GitHub - serverless-components/tencent-framework: Tencent Cloud Framework Serverless Component](https://github.com/serverless-components/tencent-framework)
* [11] [feat: enable cors · qszhu/tencent-framework@c26eef8 · GitHub](https://github.com/qszhu/tencent-framework/commit/c26eef83e285db85ac4703a15b8edaa4d1403b29)
* [12] [GitHub - serverless/components: The Serverless Framework’s new infrastructure provisioning technology — Build, compose, & deploy serverless apps in seconds…](https://github.com/serverless/components#variables-environment-variables)
* [13] [@serverless/template  -  npm](https://www.npmjs.com/package/@serverless/template)
* [14] [Documentation for environment variables does not match behavior · Issue #580 · serverless/components · GitHub](https://github.com/serverless/components/issues/580)
* [15] [GitHub - serverless/components at v1](https://github.com/serverless/components/tree/v1#environment-variables)
* [16] [tencent-koa/configure.md at v2 · serverless-components/tencent-koa · GitHub](https://github.com/serverless-components/tencent-koa/blob/v2/docs/configure.md#api-%E7%BD%91%E5%85%B3%E9%85%8D%E7%BD%AE)
* [17] [Serverless Framework 部署 Koa 框架 - 框架迁移 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/1154/40493)
