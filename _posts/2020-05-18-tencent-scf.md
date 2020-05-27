---
layout: post
title: 腾讯云云函数实践
date: 2020-05-23 22:51:00 +0800
tags: [tencent_cloud, serverless, faas, tencent_scf, wechat_mina]
---

### 1. 背景

微信小程序现可接入物流[1]，实现下单、打单和物流跟踪等功能。对于打单功能，微信提供了几种不同的接入方式，如windows only的打单软件或需要额外接入的第三方打单软件，否则需要通过`getOrder`接口获取数据自行打印[2]。`getOrder`接口会返回base64编码的运单html页面，直接在浏览器中渲染出来打印会遇到的最主要的问题是浏览器的兼容性，可通过使用可控的浏览器渲染出页面再保存为图片的方式解决。在这方面`puppeteer`[3]可以说是首选。

![快递单](/assets/images/2020-05-23/WX20200524-164924@2x.png)

*需渲染的快递单*

不过`puppeteer`也有自己的问题。比如现在的后端大多是以容器方式部署的，而配置`puppeteer`在容器中运行并不方便[4]，且配置不当可能有潜在的安全风险。恰逢最近在调研`FaaS`，就想着既然`serverless`方式部署可以不关心服务器，是不是就能转嫁掉风险了？

### 2. 问题调研

`puppeteer`运行需要`Node.js`。根据腾讯云的文档[5]，`Node.js`应用需要将依赖库一同打包上传。由于`puppeteer`依赖了`Chromium`，就意味着需要在`CentOS`下安装一个`Chromium`并打包上传，否则很容易因为环境不一致而出问题。好在腾讯云的`Node.js`环境[6]已经内置了`puppeteer`，部署时就不用再上传了。

不过在本地还是需要安装`puppeteer`，否则就无法在本地调试了。那么在部署时如何才能不上传本地安装的`puppeteer`呢？根据`serverless`的约定，只要把`puppeteer`作为`devDependencies`安装，使用`serverless`命令行部署时就会被自动忽略，不会被打包进去了。

### 3. 开发

开发没有什么难度，基本上只要把`puppeteer`和腾讯云云函数各自最简单的sample连起来就可以。值得注意的是腾讯云云函数对函数入口的形式有规定[7]，可以说是种提供商绑定，如果要解除绑定的话就需要至少多加一层抽象。

### 4. 部署

使用`serverlesss`命令行生成的默认`serverless.yml`中没有指定网关，api网关触发器的`serviceId`为空

```yaml
    events:
      - apigw:
          parameters:
            stageName: test 
            serviceId:
            httpMethod: ANY
```

这就会导致每次部署时均创建一个新的网关，访问链接也会发生变化。可以在第一次部署后把创建的网关的`serviceId`填回`serverless.yml`中（或是部署之前先自己手动在腾讯云的控制台创建一个API网关），这样就能复用这个网关，访问地址也不会变，生产环境中也可以直接用作反向代理的上游。

在线调试时依旧遇到了常见的`No usable sandbox!`错误[8]。鉴于是`serverless`环境无法配置服务器，只能以无沙盒方式启动。相信腾讯云应该有足够的安全措施🐶

### 5. 其他问题

#### 5.1 TypeScript

当前`JavaScript`后端开发基本都会采用`TypeScript`，而腾讯云的sample还都是`JavaScript`的。其实用`TypeScript`开发云函数很简单，只需确保`tsc`编译的输出目录中包含云函数所需的文件即可。通过一个简单的脚本就可以搞定：

```bash
# 编译.ts文件，输出到 dist/ 目录
$ npm run build

# 复制 package.json 和 package-lock.json
$ cp package* dist/

# 复制 serverless.yml
$ cp serverless.yml dist/

$ cd dist

# 只安装 production 依赖
$ npm ci --only=production

# serverless 部署
$ sls deploy
```

#### 5.2 使用Web框架

上面提到，由于腾讯云对于云函数的函数入口有要求，对于已有的项目可能就需要迁移，而迁移的结果可能就是提供商绑定。理想的情况下，现有的功能应该能在不同的云提供商之间无缝迁移。目前已经有些项目提供了常见的Web框架如`Express`和`koa`的支持，有时间的话会继续实践。

### 6. 小结

实际用下来的感觉，`FaaS`比较适合的场景是：

* 对响应延迟要求低
* 并发量不大
* 调用次数不多
* 无状态

像上面提到的网页转图片的服务就是这样的例子。一个企业内部可能有大量像这样的长尾服务，使用量小，使用周期短，却依旧需要和头部的服务一视同仁地进行部署和维护。如果采用`FaaS`的话开发者自己就能很方便地进行部署，从而节省大量运维资源。此外云服务商通常都提供了充沛的免费额度，这样部署和使用云函数基本就是在白嫖了。

p.s. `puppeteer`经常被用来做爬虫，而在腾讯云的文档中写着，云函数的出口IP在默认情况下是随机的[9]。这意味着什么还需要观察🐶

### 参考资料
* [1] [快递接口（商家必看） \| 微信开放文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/express/introduction.html)
* [2] [logistics.getOrder \| 微信开放文档](https://developers.weixin.qq.com/miniprogram/dev/api-backend/open-api/express/by-business/logistics.getOrder.html)
* [3] [GitHub - puppeteer/puppeteer: Headless Chrome Node.js API](https://github.com/puppeteer/puppeteer)
* [4] [puppeteer/troubleshooting.md at master · puppeteer/puppeteer · GitHub](https://github.com/puppeteer/puppeteer/blob/master/docs/troubleshooting.md#running-puppeteer-in-docker)
* [5] [云函数 依赖安装 - 开发指南 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/583/39780#node.js-.E8.BF.90.E8.A1.8C.E6.97.B6)
* [6] [云函数 Node.js 说明 - 开发指南 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/583/11060#.E7.8E.AF.E5.A2.83.E5.86.85.E7.9A.84.E5.86.85.E7.BD.AE.E5.BA.93)
* [7] [云函数 基本概念 - 开发指南 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/583/9210#.E6.89.A7.E8.A1.8C.E6.96.B9.E6.B3.95)
* [8] [puppeteer/troubleshooting.md at master · puppeteer/puppeteer · GitHub](https://github.com/puppeteer/puppeteer/blob/master/docs/troubleshooting.md#setting-up-chrome-linux-sandbox)
* [9] [云函数 通用类 - 常见问题 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/583/9180#scf-.E8.AE.BF.E9.97.AE.E5.A4.96.E7.BD.91.E6.97.B6-ip-.E6.98.AF.E9.9A.8F.E6.9C.BA.E7.9A.84.E8.BF.98.E6.98.AF.E5.9B.BA.E5.AE.9A.E7.9A.84.EF.BC.9F)
