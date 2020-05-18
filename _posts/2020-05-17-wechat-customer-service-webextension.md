---
layout: post
title:  微信客服插件（更新中）
date:   2020-05-17 20:29:15 +0800
tags: [wechat, webextension, updating]
---

### 1. 背景

### 2. 实现原理

#### 2.1 截获http请求
* https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Intercept_HTTP_requests

### 3. 实战
* 微信客服接口(https://mpkf.weixin.qq.com)
  * 手动接入(https://mpkf.weixin.qq.com/cgi-bin/kfsessionmgr)
  * 自动接入(https://mpkf.weixin.qq.com/cgi-bin/kfunaccepted?action=kfbatchcreatesession)
  * 转接(https://mpkf.weixin.qq.com/cgi-bin/kftransfer?action=gettransferinfo)

### 4. 其他问题

#### 4.1 侧边栏与页面通信
* https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Content_scripts#Communicating_with_background_scripts

#### 4.2 界面刷新
* https://github.com/Kocal/vue-web-extension

#### 4.3 多窗口时单实例
* https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Working_with_the_Tabs_API

#### 4.4 快捷回复

#### 4.5 插件发布
* https://extensionworkshop.com/documentation/enterprise/enterprise-distribution/#signed-vs-unsigned
  * 签名
    * review
  * 非签名
    * Firefox ESR

#### 4.6 支持其他浏览器
* [Chrome不支持](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/filterResponseData)
  * mitm

### 5. 小结

### 参考资料
