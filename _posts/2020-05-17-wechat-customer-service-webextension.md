---
layout: post
title: 微信客服插件
date: 2020-05-24 17:04:00 +0800
tags: [wechat, webextension]
---

### 1. 背景

微信小程序可以接入客服，通过网页版的客服工具可与用户进行对话[1]。网页版的工具只有最基础的发送消息和回复消息功能，无法与服务商的后台作进一步的整合。比如一个电商的售后客服，在用户接入后，如果能够看到该用户最近下的订单，就不必再费口舌要求用户提供订单的相关信息并在电商后台中查找订单了。在用户接入量大时可极大地提升客服效率。

微信小程序提供了消息推送接口[2]，可在服务端接收和发送客服消息。理论上利用推送消息中的用户信息，就可以在电商后台中找到对应用户的信息并展示出来，但这样就需要重新开发一个客服工具了。有没有办法直接修改网页版客服工具来整合电商后台呢？

### 2. 问题分析

在浏览器的开发者工具中可以看到，当手动接入一个新用户时，浏览器会向以下地址发起`GET`请求:

```
https://mpkf.weixin.qq.com/cgi-bin/kfsessionmgr
```

返回值类似这样：

```javascript
{
  "base_resp": {
    "ret": 0,
    "err_msg": "OK"
  },
  "kfcreatesession_resp": [
    {
      "base_resp": {
        "ret": 0,
        "err_msg": "OK"
      },
      "kfsession": {
        // ...
        "nickname": "xxx",
        "headimgurl": "http://wx.qlogo.cn/mmhead/abcdefghijklmnopqrstuvwxyz",
        "fans_openid": "zyxwvutsrqponmlkjihgfedcba"
      },
      "msgsummary": {
        "msg": {
          // ...
        },
        // ...
      }
    }
  ]
}
```

其中`kfcreatesession_resp.kfsession.fans_openid`就是用户在小程序中的openid，是用户在小程序中的唯一标识，电商后台可通过这个标识找到该用户的相关信息。所以只需在接入用户时截获这个请求，就能从请求响应中抽取出用户的唯一标识，继而获得用户的相关信息。

接下来的问题是，如何在接入多个用户时只显示当前选中用户的相关信息？在绑定了页面的click事件后，发现event target中能用于区分用户的信息就只有用户头像的url了。好在用户接入的请求响应中也包含了用户头像的url，所以只需要维护一个Map，key是用户头像的url，value是用户的openid，就可以在点击了用户会话之后获得用户的唯一标识，进而展示用户的相关信息了。

修改网页界面通常可通过user script进行。但在这个问题中，由于需要截获http请求，就需要通过有更高权限的浏览器插件。使用插件开发的另一个好处是可以把用户相关信息显示在浏览器侧边栏中，这样就不会把原有的页面搞乱。

Mozilla的文档[3]中给出了在浏览器插件中截获http请求的具体方法。

### 3. 开发

实际开发的过程并非一帆风顺。比如

* 用户会话中的用户头像url是这样的形式：

```
https://wx.qlogo.cn/mmhead/abcdefghijklmnopqrstuvwxyz/132
```

而截获的请求响应中的url是这样的：

```
http://wx.qlogo.cn/mmhead/abcdefghijklmnopqrstuvwxyz
```

形式上有些差别，所以不能直接作为key去Map中查找。

* 根据浏览器插件的规范，截获请求需要在后台脚本中执行，而显示信息是在侧边栏中，这就需要让双方进行通信。对此文档中也有详细介绍[4]。

* 切换用户会话后需要刷新侧边栏显示当前用户的信息，这里选择了重新创建并替换DOM元素。要使用一些现代的前端框架如`vue`或`react`应该也是可以的，之后有时间可以尝试下。

* 在实现了显示手动接入的用户信息之后，发现对于自动接入的用户无效。从开发者工具中看到，对于自动接入的用户，浏览器会请求另一个地址：

```
https://mpkf.weixin.qq.com/cgi-bin/kfunaccepted?action=kfbatchcreatesession
```

* 在实现了显示自动接入的用户信息之后，又发现对于客服转接的用户无效。从开发者工具中看到，此时浏览器会请求：

```
https://mpkf.weixin.qq.com/cgi-bin/kftransfer?action=gettransferinfo
```

好在响应的格式都差不多，所以实现起来只是多加几个需要截获的url而已。

![插件截图](/assets/images/1590310867.jpg)

*插件截图*

### 4. 其他问题

至此插件就可以正常运行了，但还可以做些改进

* 如果开了多个浏览器窗口，则每个浏览器窗口都可以打开侧边栏，但能够和后台脚本通信的端口只需要一个。所以需要加入额外的判断，看当前的浏览器窗口中是否打开了目标页面。这需要使用浏览器插件的标签API[5]。

* 有些常用的回复可以做成一键发送。在侧边栏中点击对应回复的链接，就把回复内容自动插入到当前用户的对话框中，一按回车键就能发送。

* 插件的发布比较繁琐，需经过签名和代码审核。而安装未签名的插件的最简单的方式，是安装Firefox ESR版本[6]。

* 这个方案实现的核心是能够截获http响应的内容，而这个功能似乎只有Firefox插件才能做到[7]。若要支持其他浏览器，可能需要在本地开一个中间人代理来截获请求，并与浏览器进行通信。这样部署起来就很麻烦，代价接近重新开发一个客服客户端了。

### 5. 小结

微信的客服工具网页版是`react`开发的，新消息是通过轮询获得的，偶尔会遇到丢消息或发不出消息的情况。有条件的话还是推荐通过接入消息推送的方式自行开发客服客户端并与后台整合。不过微信小程序的消息推送接口也没设计好，除了客服消息推送以外，还会推送物流跟踪信息[8]。而小程序的后台只支持配置一个消息推送接口，微信小程序力推的云开发也仅支持客服消息[2]。所以如果既要支持客服消息又要支持物流跟踪消息的话，就需要自行设计和开发消息分发机制了。

### 参考资料
* [1] [客服消息使用指南 | 微信开放文档](https://developers.weixin.qq.com/miniprogram/introduction/custom.html#%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%B9%B3%E5%8F%B0%E7%BD%91%E9%A1%B5%E7%89%88%E5%AE%A2%E6%9C%8D%E5%B7%A5%E5%85%B7%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)
* [2] [消息推送 | 微信开放文档](https://developers.weixin.qq.com/miniprogram/dev/framework/server-ability/message-push.html)
* [3] [Intercept HTTP requests - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Intercept_HTTP_requests)
* [4] [Content scripts - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Content_scripts#Communicating_with_background_scripts)
* [5] [Working with the Tabs API - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Working_with_the_Tabs_API)
* [6] [Enterprise distribution | Firefox Extension Workshop](https://extensionworkshop.com/documentation/enterprise/enterprise-distribution/#signed-vs-unsigned)
* [7] [webRequest.filterResponseData() - Mozilla | MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest/filterResponseData)
* [8] [logistics.onPathUpdate | 微信开放文档](https://developers.weixin.qq.com/miniprogram/dev/api-backend/open-api/express/by-business/logistics.onPathUpdate.html)
