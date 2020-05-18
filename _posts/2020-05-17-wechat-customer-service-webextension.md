---
layout: post
title:  微信客服插件（更新中）
date:   2020-05-17 20:29:15 +0800
tags: [wechat, webextension, updating]
---

* 浏览器
  * Firefox
    * [截获http请求](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Intercept_HTTP_requests)
  * [Chrome不支持](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/filterResponseData)
* 微信客服接口
  * 手动接入(https://mpkf.weixin.qq.com/cgi-bin/kfsessionmgr)
  * 自动接入(https://mpkf.weixin.qq.com/cgi-bin/kfunaccepted?action=kfbatchcreatesession)
  * 转接(https://mpkf.weixin.qq.com/cgi-bin/kftransfer?action=gettransferinfo)
* 其他问题
  * 侧边栏与页面通信
  * 界面刷新
  * 多窗口时单实例
  * 快捷回复
  * 插件发布
    * 签名
      * review
    * 非签名
      * Firefox ESR
  * 支持其他浏览器
    * mitm