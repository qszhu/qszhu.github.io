---
layout: post
title: 迁移已有项目到腾讯serverless服务（更新中）
date: 2020-05-31 20:00:00 +0800
tags: [tencent_cloud, serverless, updating]
---

* component开发
  * `serverless.yml`: component绝对路径
  * 删除`~/.serverless/cache`?
* 兼容性问题
  * import模块
    * koa component需要名为app的commonjs模块: `require('./app')`
    * typescript输出es module而非cjs
      * ts -(tsc)-> es -(babel)-> cjs 
    * 默认export
      * `export default`输出为`exports.default`
      * `export =`输出为`module.exports`
      * https://github.com/microsoft/TypeScript/issues/2719
  * 日志
    * 不支持`process.stderr`
    * `debug`包无法使用
      * `debug.log=console.log.bind(console)`
  * CORS支持
    * api网关
* 其他问题
  * 私密`.env`
    * 写在`serverless.yml`中